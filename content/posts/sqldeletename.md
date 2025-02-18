+++ 
draft = false
date = 2025-02-18T11:49:27+08:00
title = "在 MySQL 中删除重复的记录，但保留其中一条"
description = "本方案介绍了如何在 MySQL 中删除重复的记录，同时保留其中一条，避免数据冗余的同时保证数据完整性。"
slug = "在-mysql中删除重复的记录-但保留其中一条"
authors = ["FindList"]
tags = ["MySQL", "重复记录", "数据清理", "SQL"]
categories = ["技术方案"]
externalLink = ""
series = ["技术文章系列"]
+++
## 删除重复的记录，保留一条

如果你想删除重复的记录，但保留其中一条，可以通过以下步骤实现：

首先，你可以通过一个子查询找出那些重复的 `incident_name`。
然后，使用 `DELETE` 语句删除重复的记录。
假设每一行都有一个唯一的标识符 `id`（通常是自增的），你可以按照以下方法操作：

### 方案：删除重复的记录，保留一条

### SQL 语句解释

这段 SQL 语句的目的是删除表 `knowledge_security_incident` 中重复的记录，只保留每个 `incident_name` 的最早一条记录。具体步骤如下：

1. **子查询**：
    - `SELECT MIN(id)`: 找出每组 `incident_name` 中最小的 `id`，即保留最早插入的那一条记录。
    - `FROM knowledge_security_incident`: 从 `knowledge_security_incident` 表中查询数据。
    - `WHERE incident_name != ''`: 只考虑 `incident_name` 不为空的记录。
    - `GROUP BY incident_name`: 按照 `incident_name` 分组，找到每个重复组中的第一条记录。

2. **主查询**：
    - `DELETE FROM knowledge_security_incident`: 删除 `knowledge_security_incident` 表中的记录。
    - `WHERE id NOT IN (...)`: 删除所有 `id` 不在子查询结果中的记录，即删除重复的记录。

### 结果

这样做可以确保对于每个 `incident_name`，只保留其中一条记录，其他重复的记录会被删除。

### 注意事项

- **备份数据**：在执行删除操作前，最好先备份数据，以防误删重要数据。
- **调整逻辑**：如果需要根据其他条件保留某些记录，可以调整 `SELECT MIN(id)` 中的逻辑。