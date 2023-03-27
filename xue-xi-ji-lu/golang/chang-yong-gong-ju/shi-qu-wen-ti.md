# 时区问题

开发环境使用 CST时区

测试环境使用 UTC时区

数据库使用 UTC时区

采用gorm 框架

## 故障现象

本地开发环境和线上测试环境使用相同的数据库，结果通过开发环境创建的数据在测试环境读取时间异常

## 排查过程

通过断点排查，发现是运行环境时区问题

1. 查看数据库时区，使用UTC时区。关于问题时间记录，采用的 timestamp 格式记录 。
2. 无论数据库时区设置如何，gorm 获取时候可以设置时区

dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4\&parseTime=True&**loc=Local**"

db, err := gorm.Open(mysql.Open(dsn), \&gorm.Config{})

### 使用 time.Now() 插入并查询

loc=Local

dev time: 2022-03-01T14:10:10+08:00

from database record: {2022-03-01 14:10:10 +0800 CST 2022-03-01 14:10:10 +0800 CST}

database record: 2022-03-01 14:10:10 | 2022-03-01 14:10:10

loc=UTC

dev time: 2022-03-01T14:08:36+08:00

from database record: {2022-03-01 06:08:37 +0000 UTC 2022-03-01 06:08:37 +0000 UTC}

database record: 2022-03-01 06:08:37 | 2022-03-01 06:08:37

### 使用 time.Now().UTC 插入并查询

loc=Local

dev time: 2022-03-01T06:17:13Z

from database record: {2022-03-01 14:17:14 +0800 CST 2022-03-01 14:17:14 +0800 CST}

database record: 2022-03-01 14:17:14 | 2022-03-01 14:17:14

loc=UTC

dev time: 2022-03-01T06:19:10Z

from database record: {2022-03-01 06:19:11 +0000 UTC 2022-03-01 06:19:11 +0000 UTC}

database record: 2022-03-01 06:19:11 | 2022-03-01 06:19:11

## 小结

1. 保存时，GORM 根据 loc 属性，将time.Time 时间转成 loc 时区
2. 数据库保存时候未能保留时区，GORM 提供转换时区后的 time.Time 时间部分
3. 读取时，GORM根据 loc 属性，将数据库读取的时间强制添加时区

按照上述结论，目前数据异常问题在于

1. 数据创建时，程序本地时区为CST，并采用 loc=Local，存放在数据库的为 0+8 的时间
2. 数据读取时，程序本地时区为UTC，并采用loc=Local，读取的数据库时间为 8+0 时间，再转成CST时间变成 8+8
3. 最终显示从8点变成16点
