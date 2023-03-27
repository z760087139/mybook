# 锁

> 查看当前事务级别
>
> ```sql
> show variables like '%tx_isolation%'
> ```

### 术语

#### 脏读

脏读指的是读到了其他事务未提交的数据，未提交意味着这些数据可能会回滚，也就是可能最终不会存到数据库中，也就是不存在的数据。读到了并一定最终存在的数据，这就是脏读。

#### 可重复读

可重复读指的是在一个事务内，最开始读到的数据和事务结束前的任意时刻读到的同一批数据都是一致的。通常针对数据\*\*更新（UPDATE）\*\*操作。

#### 不可重复读

对比可重复读，不可重复读指的是在同一事务内，不同的时刻读到的同一批数据可能是不一样的，可能会受到其他事务的影响，比如其他事务改了这批数据并提交了。通常针对数据\*\*更新（UPDATE）\*\*操作。

#### 幻读

幻读是针对数据\*\*插入（INSERT）\*\*操作来说的。假设事务A对某些行的内容作了更改，但是还未提交，此时事务B插入了与事务A更改前的记录相同的记录行，并且在事务A提交之前先提交了，而这时，在事务A中查询，会发现好像刚刚的更改对于某些数据未起作用，但其实是事务B刚插入进来的，让用户感觉很魔幻，感觉出现了幻觉，这就叫幻读。

### 事务隔离级别

#### Read Uncommitted

可以读取其他事务未提交的内容（脏读）

#### Read Committed

可以读取其他事务已提交的内容，避免脏读，存在不可重复读

#### Repeatable Read (Mysql 默认级别)

同一事务内的任意时刻的数据内容一致，但是无法避免其他事务对表内容进行插入动作导致的幻读

#### Serializable

串行执行事务，同时解决了脏读、不可重复读、幻读问题，但是存在性能影响

| 隔离级别  | 脏读 | 不可重复读 | 幻读 |
| ----- | -- | ----- | -- |
| 读未提交  | 是  | 是     | 是  |
| 不可重复读 | 否  | 是     | 是  |
| 可重复读  | 否  | 否     | 是  |
| 串行化   | 否  | 否     | 否  |

### 锁类型

#### **共享锁**(S锁)

假设事务T1对数据A加上共享锁，那么事务T2**可以**读数据A，**不能**修改数据A。

#### **排他锁**(X锁)

假设事务T1对数据A加上共享锁，那么事务T2**不能**读数据A，**不能**修改数据A。 我们通过`update`、`delete`等语句加上的锁都是行级别的锁。只有`LOCK TABLE … READ`和`LOCK TABLE … WRITE`才能申请表级别的锁。

#### **意向共享锁**(IS锁)

一个事务在获取（任何一行/或者全表）S锁之前，一定会先在所在的表上加IS锁。

#### **意向排他锁**(IX锁)

一个事务在获取（任何一行/或者全表）X锁之前，一定会先在所在的表上加IX锁

### mysql 加锁算法

#### Record Lock

行锁，对索引行进行加锁。innodb 存在聚簇索引，最终都会在聚簇索引上加锁

#### Gap Lock

间隙锁，**只有RR/ Serializable**才会存在

### 测试记录

```sql
# ddl
-- auto-generated definition
create table t1
(
    id   int auto_increment
        primary key,
    name varchar(20) null,
    type varchar(20) null
);
insert into t1(name,type) value('a','ss');
```

#### RR级别 + for update + 无索引

```sql
# session 1
begin;
select * from t1 where name = "a" for update;
```

```sql
# seesion 3
# 5.7
select * from information_schema.innodb_locks;
# 8.0
select * from performance_schema.data_locks;
select * from performance_schema.data_lock_waits;
# select THREAD_ID,OBJECT_NAME,INDEX_NAME,LOCK_TYPE,LOCK_MODE,LOCK_STATUS,LOCK_DATA from performance_schema.data_locks;
```

| THREAD\_ID | OBJECT\_NAME | INDEX\_NAME | LOCK\_TYPE | LOCK\_MODE | LOCK\_STATUS | LOCK\_DATA             |
| ---------- | ------------ | ----------- | ---------- | ---------- | ------------ | ---------------------- |
| 50         | t1           | NULL        | TABLE      | IX         | GRANTED      | NULL                   |
| 50         | t1           | PRIMARY     | RECORD     | X          | GRANTED      | supremum pseudo-record |
| 50         | t1           | PRIMARY     | RECORD     | X          | GRANTED      | 1                      |

执行 session 1 的请求后，查看锁信息，发现出现 record lock 同时还有 table IX

```sql
# seesion 2
begin;
select * from t1 where name = "b"; # no lock
select * from t1 where name = 'a'; # no lock
select * from t1 where name = "b" for update; # lock
insert into t1(name,type) value('a','ss'); # lock
```

#### RR级别 + for update + name 索引

```sql
# index ddl
create index t1_ind on t1(name);
```

```sql
# session 1
begin;
select * from t1 where name = "a" for update;
```

```sql
# session 2
begin;
select * from t1 where name = 'b' for update; # no lock
commit;
```

| THREAD\_ID | OBJECT\_NAME | INDEX\_NAME | LOCK\_TYPE | LOCK\_MODE      | LOCK\_STATUS | LOCK\_DATA             |
| ---------- | ------------ | ----------- | ---------- | --------------- | ------------ | ---------------------- |
| 68         | t1           | NULL        | TABLE      | IX              | GRANTED      | NULL                   |
| 68         | t1           | t1\_ind     | RECORD     | X               | GRANTED      | supremum pseudo-record |
| 50         | t1           | NULL        | TABLE      | IX              | GRANTED      | NULL                   |
| 50         | t1           | t1\_ind     | RECORD     | X               | GRANTED      | supremum pseudo-record |
| 50         | t1           | t1\_ind     | RECORD     | X               | GRANTED      | 'a', 1                 |
| 50         | t1           | PRIMARY     | RECORD     | X,REC\_NOT\_GAP | GRANTED      | 1                      |

增加索引后，锁信息及主键的锁出现了变化

主键的锁类型变成了 X,REC\_NOT\_GAP，无间隙锁

supremum pseudo-record 不再在主键上，变成索引上加锁
