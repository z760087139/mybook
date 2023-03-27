# GORM规范

## Model 结构

**优先使用自定义表名或者动态表名方式**，显式表示结构预期对应表名

#### 默认表名

​ gorm 会将匿名结构采用 蛇形命名（下划线分割）作为表名使用

```go
// /Users/hanhui/go/pkg/mod/gorm.io/gorm@v1.21.9/gorm.go:140
	if config.NamingStrategy == nil {
		config.NamingStrategy = schema.NamingStrategy{}
	}
// /Users/hanhui/go/pkg/mod/gorm.io/gorm@v1.21.9/schema/naming.go:29
// NamingStrategy tables, columns naming strategy
type NamingStrategy struct {
	TablePrefix   string
	SingularTable bool
	NameReplacer  Replacer
	NoLowerCase   bool
}

// TableName convert string to table name
func (ns NamingStrategy) TableName(str string) string {
	if ns.SingularTable {
		return ns.TablePrefix + ns.toDBName(str)
	}
	return ns.TablePrefix + inflection.Plural(ns.toDBName(str))
}
```

#### 自定义表名

```go
type Tabler interface {
    TableName() string
}
// TableName 会将 User 的表名重写为 `profiles`
func (User) TableName() string {
  return "profiles"
}
```

​ 通过对 model 实现 Tabler interface ，可以自定义表的命名。但是这种方式不支持动态变化

#### 动态表名

```go
func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
  return func (tx *gorm.DB) *gorm.DB {
    if user.Admin {
      return tx.Table("admin_users")
    }

    return tx.Table("users")
  }
}

db.Scopes(UserTable(user)).Create(&user)
```

#### 临时表名

```go
// 根据 User 的字段创建 `deleted_users` 表
db.Table("deleted_users").AutoMigrate(&User{})
```

## Logger

​ logger 采用公共日志库中实现的 log.DBLogger

```go
type Interface interface {
   LogMode(LogLevel) Interface
   Info(context.Context, string, ...interface{})
   Warn(context.Context, string, ...interface{})
   Error(context.Context, string, ...interface{})
   Trace(ctx context.Context, begin time.Time, fc func() (string, int64), err error)
}

// util.log 已有 db logger 的实现
// 全局模式
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  // 引用 log.DBLogger
  Logger: log.DBLogger,
})
// 新建会话模式
tx := db.Session(&Session{Logger: newLogger})
// 日志级别
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: logger.Default.LogMode(logger.Silent),
})

```

## 事务

​ 事务建议采用模式2，更简洁并且减少闭包可能导致的变量引用问题。但**必须**调用rollback / commit 方法结束事务。

#### 模式1

```go
db.Transaction(func(tx *gorm.DB) error {
  // 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    // 返回任何错误都会回滚事务
    return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    return err
  }
  // 嵌套事务
  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user2)
    return errors.New("rollback user2") // Rollback user2
  })
})
```

#### 模式2

```go
// 开始事务
tx := db.Begin()
// 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
tx.Create(...)
// ...
// 遇到错误时回滚事务
tx.Rollback()
// 否则，提交事务
tx.Commit()
```

### 注意事项

​ 该事务形式为**单次 data 层的调用**。

​ 如果涉及分布式事务或者涉及 biz 层的业务逻辑，**不建议**将 tx （事务）放置 biz 层并传入 data层实现事务（example 1)。

​ _example 2，example 3仅作参考_，需按照实际情况进行分布式事务设计。

#### 原因

​ biz 层不应该关注 data层的实现内容，不符合依赖倒置的要求。同时传入 tx 意味着 biz 要求 data 必须引入 grom.DB，但实际 data 并不要求采用 gorm.DB。

**example 1**

```go
// biz.go
func Biz(){
  tx := db.Begin()
  // 业务...
  data.Update1(tx, a)
  // 业务...
  data.Update2(tx, b)
  tx.Commit
}
// data.go
func Update1(tx gorm.DB, a struct){
  tx.Update(...)
}
func Update2(tx gorm.DB, b struct){
  tx.Update(...)
}
```

​ 上述的业务，建议按照实际情况，进行数据库操作合并或者data层提供回滚动作，并在biz 层调用。

**example 2**

```go
// biz.go
func Biz(){
  // ... 业务内容
  // 将结果缓存在内存
  data.UpdateAll(c struct)
}

// data.go
func UpdateAll(){
  tx := db.Begin()
  // 相当于 update 1
  a := conv(c)
  tx.Update(a)
  // 相当于 update 2
  b := conv(c)
  tx.Update(b)
  tx.Commit()
}
```

**example 3**

```go
// biz.go
func Biz(){
   tx := db.Begin()
  // 业务...
  data.Update1(tx, a)
  defer func(){
    if err != nil {
      data.RollBack1()
    }
  }
  // 业务...
  data.Update2(tx, b)
  defer func(){
    if err != nil {
      data.RollBack2()
    }
  }
  tx.Commit
}
```

### SavePoint、RollbackTo

​ GORM 提供了 `SavePoint`、`Rollbackto` 方法，来提供保存点以及回滚至保存点功能（同上面说明，只在单次 data 层调用实现），例如：

```
tx := db.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // Rollback user2

tx.Commit() // Commit user1
```

## 创建

### 用指定的字段创建记录

​ 创建记录并更新给出的字段。

```
db.Select("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`name`,`age`,`created_at`) VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")
```

​ 创建一个记录且一同忽略传递给略去的字段值。

```
db.Omit("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`birthday`,`updated_at`) VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")
```

### 批量插入

​ 要有效地插入大量记录，请将一个 `slice` 传递给 `Create` 方法。 GORM 将生成单独一条SQL语句来插入所有数据，并回填主键的值，钩子方法也会被调用。

```
var users = []User{{Name: "jinzhu1"}, {Name: "jinzhu2"}, {Name: "jinzhu3"}}
db.Create(&users)

for _, user := range users {
  user.ID // 1,2,3
}
```

## 更新

​ 更新分为部分更新(PATCH)与全量更新(PUT)两种动作。

​ 一般情况下，**建议采用部分更新**

### 部分更新

```go
db.Model(&user).Updates(User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
```

当使用 `struct` 更新时，**默认情况下，GORM 只会更新非零值的字段**

### 全部更新

您想要在更新时选定、忽略某些字段，您可以使用 `Select`、`Omit`

```go
// 使用 Struct 进行 Select（会 select 零值的字段）
db.Model(&user).Select("Name", "Age").Updates(User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;

// Select 所有字段（查询包括零值字段的所有字段）
db.Model(&user).Select("*").Update(User{Name: "jinzhu", Role: "admin", Age: 0})

// Select 除 Role 外的所有字段（包括零值字段的所有字段）
db.Model(&user).Select("*").Omit("Role").Update(User{Name: "jinzhu", Role: "admin", Age: 0})
```

#### 批量更新

默认设置 GORM 禁止不带WHERE条件的批量更新

```go
db.Model(&User{}).Where("1 = 1").Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu" WHERE 1=1
```

## 查询

​ 只有在目标 struct（即 \&users) **是指针或者通过 `db.Model()` 指定 model 时**，该方法才有效

```go
// Get all records
result := db.Find(&users)
// SELECT * FROM users;
```

\*\*\* 禁止使用 `select *` 查询\*\*\* ，即使需要返回查询的全部内容，都必须使用select 指明列名

​ 复杂度较高的SQL可以考虑采用 Raw 方法查询。

​ **建议**gorm框架查询逻辑都注释预期语句及查询效果。

​ 关于 Join/ Where / Or / Not 具体逻辑及用法建议查看官网例子

### 分页查询

```go
db.Limit(10).Offset(5).Find(&users)
// SELECT * FROM users OFFSET 5 LIMIT 10;
```

## 删除

​ **建议优先采用逻辑删除**，逻辑删除可以将数据迁移至回收表，减少查询时候的状态条件过滤。具体根据业务要求设计。

## 错误代码

```go
// https://github.com/go-gorm/gorm/blob/master/errors.go	
var (
	// ErrRecordNotFound record not found error
	ErrRecordNotFound = logger.ErrRecordNotFound
	// ErrInvalidTransaction invalid transaction when you are trying to `Commit` or `Rollback`
	ErrInvalidTransaction = errors.New("invalid transaction")
	// ErrNotImplemented not implemented
	ErrNotImplemented = errors.New("not implemented")
	// ErrMissingWhereClause missing where clause
	ErrMissingWhereClause = errors.New("WHERE conditions required")
	// ErrUnsupportedRelation unsupported relations
	ErrUnsupportedRelation = errors.New("unsupported relations")
	// ErrPrimaryKeyRequired primary keys required
	ErrPrimaryKeyRequired = errors.New("primary key required")
	// ErrModelValueRequired model value required
	ErrModelValueRequired = errors.New("model value required")
	// ErrInvalidData unsupported data
	ErrInvalidData = errors.New("unsupported data")
	// ErrUnsupportedDriver unsupported driver
	ErrUnsupportedDriver = errors.New("unsupported driver")
	// ErrRegistered registered
	ErrRegistered = errors.New("registered")
	// ErrInvalidField invalid field
	ErrInvalidField = errors.New("invalid field")
	// ErrEmptySlice empty slice found
	ErrEmptySlice = errors.New("empty slice found")
	// ErrDryRunModeUnsupported dry run mode unsupported
	ErrDryRunModeUnsupported = errors.New("dry run mode unsupported")
	// ErrInvalidDB invalid db
	ErrInvalidDB = errors.New("invalid db")
	// ErrInvalidValue invalid value
	ErrInvalidValue = errors.New("invalid value, should be pointer to struct or slice")
	// ErrInvalidValueOfLength invalid values do not match length
	ErrInvalidValueOfLength = errors.New("invalid association values, length doesn't match")
	// ErrPreloadNotAllowed preload is not allowed when count is used
	ErrPreloadNotAllowed = errors.New("preload is not allowed when count is used")
)
```
