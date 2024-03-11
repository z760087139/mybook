---
description: 记录常用但比较特殊的mysql 语句
---

# 常用MySQL 语法

### json search

json\_extract( "json", "$.name")

### ascii to utf8

convert( "ascii" using utf8)

#### group and latest

按照name分组，再使用 id 排序，拿第一条数据

```sql
WITH ranked_messages AS (
  SELECT m.*, ROW_NUMBER() OVER (PARTITION BY name ORDER BY id DESC) AS rn
  FROM messages AS m
)
SELECT * FROM ranked_messages WHERE rn = 1;
```

