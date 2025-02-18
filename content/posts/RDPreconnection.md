+++
draft = false
date = 2025-02-17T10:49:27+08:00
title = "解决 Guacamole RDP 连接按键循环导致内存卡死的完整方案"
description = "本方案提供了详细的步骤，帮助解决 Guacamole RDP 连接时出现按键循环问题，防止内存崩溃卡死。"
slug = "解决-Guacamole-RDP-连接按键循环导致内存卡死的完整方案"
authors = ["FindList"]
tags = ["Guacamole", "RDP", "按键循环", "内存问题", "解决方案"]
categories = ["技术方案"]
externalLink = ""
series = ["技术文章系列"]
+++


# 解决 RDP 连接后按键循环发送导致内存卡死问题

为了解决 RDP 连接后按键循环发送导致内存卡死的问题，主要需要确保在重新连接时正确清理旧的 Guacamole 客户端及其事件监听器。

## 1. 使用 `useRef` 跟踪客户端实例
将 `client`、`keyboard` 等对象使用 `useRef` 保存，以便在清理时获取最新实例。

## 2. 在重新连接前清理旧资源
在创建新客户端前，断开旧连接并移除事件监听器。

## 3. 确保事件处理函数使用最新实例
在键盘事件处理函数中，使用 `ref.current` 来引用当前客户端。主要集中在正确管理 Guacamole 客户端生命周期和事件监听器清理。

## 代码示例

```javascript
import React, { useEffect, useRef, useState } from "react";
// ... 其他导入保持不变

const GuacdConsumer = () => {
    // 使用 useRef 替代 useState 保存易变对象
    const clientRef = useRef(null);
    const keyboardRef = useRef(null);
    const mouseRef = useRef(null);
    const touchRef = useRef(null);
    const tunnelRef = useRef(null);

    const renderDisplay = (sessionId, protocol, width, height, value) => {
        // 清理旧连接
        const cleanup = () => {
            if (clientRef.current) {
                clientRef.current.disconnect();
                clientRef.current = null;
            }
            if (tunnelRef.current) {
                tunnelRef.current.disconnect();
                tunnelRef.current = null;
            }
            if (keyboardRef.current) {
                keyboardRef.current.onkeydown = null;
                keyboardRef.current.onkeyup = null;
                keyboardRef.current = null;
            }
            if (mouseRef.current) {
                mouseRef.current.onmousedown = null;
                mouseRef.current.onmouseup = null;
                mouseRef.current.onmousemove = null;
                mouseRef.current = null;
            }
            if (touchRef.current) {
                touchRef.current.onmousedown = null;
                touchRef.current.onmousemove = null;
                touchRef.current.onmouseup = null;
                touchRef.current = null;
            }
            const displayEle = document.getElementById("display");
            if (displayEle) displayEle.innerHTML = "";
        };

        cleanup(); // 先清理旧连接

        let tunnel = new Guacamole.WebSocketTunnel(
            `${wsServer}/sessions/${sessionId}/tunnel`
        );
        tunnelRef.current = tunnel;

        let client = new Guacamole.Client(tunnel);
        clientRef.current = client;

        client.onclipboard = handleClipboardReceived;
        client.onstatechange = (state) => {
            if (state === 3) {
                message.destroy("login");
                setModalOpen(false);
            }
            onClientStateChange(state, sessionId, tunnel);
        };

        client.onerror = onError;
        tunnel.onerror = onError;

        const displayEle = document.getElementById("display");
        const element = client.getDisplay().getElement();
        displayEle.appendChild(element);

        client.connect(paramStr);

        const display = client.getDisplay();
        display.onresize = function (width, height) {
            display.scale(
                Math.min(
                    window.innerHeight / display.getHeight(),
                    window.innerWidth / display.getHeight()
                )
            );
        };

        const sink = new Guacamole.InputSink();
        displayEle.appendChild(sink.getElement());
        sink.focus();

        const keyboard = new Guacamole.Keyboard(sink.getElement());
        keyboardRef.current = keyboard;

        const handleKeyDown = (keysym) => {
            if (!clientRef.current) return;
            clientRef.current.sendKeyEvent(1, keysym);
        };

        const handleKeyUp = (keysym) => {
            if (clientRef.current) {
                clientRef.current.sendKeyEvent(0, keysym);
            }
        };

        keyboard.onkeydown = handleKeyDown;
        keyboard.onkeyup = handleKeyUp;

        const sinkFocus = debounce(() => {
            sink.focus();
        });

        const mouse = new Guacamole.Mouse(element);
        mouseRef.current = mouse;

        mouse.onmousedown = mouse.onmouseup = function (mouseState) {
            sinkFocus();
            clientRef.current?.sendMouseState(mouseState);
        };

        mouse.onmousemove = function (mouseState) {
            sinkFocus();
            clientRef.current?.getDisplay().showCursor(false);
            mouseState.x = mouseState.x / display.getScale();
            mouseState.y = mouseState.y / display.getScale();
            clientRef.current?.sendMouseState(mouseState);
        };

        const touch = new Guacamole.Mouse.Touchpad(element);
        touchRef.current = touch;

        touch.onmousedown =
        touch.onmousemove =
        touch.onmouseup =
        function (state) {
            clientRef.current?.sendMouseState(state);
        };
    };

    useEffect(() => {
        return () => {
            // 组件卸载时彻底清理
            if (clientRef.current) {
                clientRef.current.disconnect();
            }
            if (tunnelRef.current) {
                tunnelRef.current.disconnect();
            }
            if (keyboardRef.current) {
                keyboardRef.current.onkeydown = null;
                keyboardRef.current.onkeyup = null;
            }
            if (mouseRef.current) {
                mouseRef.current.onmousedown = null;
                mouseRef.current.onmouseup = null;
                mouseRef.current.onmousemove = null;
            }
            if (touchRef.current) {
                touchRef.current.onmousedown = null;
                touchRef.current.onmousemove = null;
                touchRef.current.onmouseup = null;
            }
        };
    }, []);

    const sendCombinationKey = (keys) => {
        const client = clientRef.current;
        if (!client) return;
        
        requestAnimationFrame(() => {
            keys.forEach(keysym => client.sendKeyEvent(1, keysym));
            keys.forEach(keysym => client.sendKeyEvent(0, keysym));
        });
    };
};

## 主要改动说明

### 1. 使用 `useRef` 管理易变对象
- 将 `client`、`tunnel` 等 Guacamole 对象改为 `useRef` 存储，确保在闭包中始终访问最新实例。

### 2. 连接前清理机制
- 在创建新连接前增加 `cleanup` 函数，彻底清理旧连接相关资源。

### 3. 严格的事件监听清理
- 在组件卸载和重新连接时，明确移除所有事件监听器。

### 4. 安全的事件处理
- 所有事件处理函数都通过 `clientRef.current` 访问客户端，并增加空值检查。

### 5. 动画帧优化
- 在发送组合键时使用 `requestAnimationFrame` 避免快速连续操作。

## 这些修改可以确保：
- 每次重新建立连接时旧连接会被正确清理。
- 不会存在多个客户端实例同时。
- 键盘、鼠标事件监听器被正确清理，防止重复触发问题。