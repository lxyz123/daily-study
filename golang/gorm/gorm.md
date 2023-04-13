## 特性
### v2 vs v1

- 性能改进
- 代码模块化
- Context，批量插入，预编译模式，DryRun 模式，Join 预加载，Find To Map，Create From Map，FindInBatches 支持
- 支持嵌套事务，SavePoint，Rollback To SavePoint
- SQL 生成器，命名参数，分组条件，Upsert，锁， 支持 Optimizer/Index/Comment Hint，子查询改进，使用 SQL 表达式、Context Valuer 进行 CRUD
- 支持完整的自引用，改进 Join Table，批量数据的关联模式
- 允许多个字段用于追踪 create、update 时间 ，支持 UNIX （毫 / 纳）秒
- 支持字段权限：只读、只写、只创建、只更新、忽略
- 新的插件系统，为多个数据库提供了官方插件，读写分离，prometheus 集成…
- 全新的 Hook API：带插件的统一接口
- 全新的 Migrator：允许为关系创建数据库外键，更智能的 AutoMigrate，支持约束、检查器，增强索引支持
- 全新的 Logger：支持 context、改进可扩展性
- 统一命名策略：表名、字段名、连接表名、外键、检查器、索引名称规则
- 更好的自定义类型支持（例如： JSON）
## 创建
### 创建记录
```go
type User struct{
    Name `type i<>`
}
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

result := db.Create(&user) // 通过数据的指针来创建

user.ID             // 返回插入数据的主键
result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```
### 用指定的字段创建记录
#### 创建记录并更新给出的字段
```go
db.Select("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`name`,`age`,`created_at`) VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")

```
#### 创建一个记录并一同忽略传递给略去的字段
```go
db.Omit("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`birthday`,`updated_at`) VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")
```
### 批量插入
```go
ar users = []User{{name: "jinzhu_1"}, ...., {Name: "jinzhu_10000"}}

// 数量为 100
db.CreateInBatches(users, 100)
```
### 创建钩子
```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  u.UUID = uuid.New()

    if u.Role == "admin" {
        return errors.New("invalid role")
    }
    return
}
```
跳过钩子方法
```go
DB.Session(&gorm.Session{SkipHooks: true}).Create(&user)

DB.Session(&gorm.Session{SkipHooks: true}).Create(&users)

DB.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
```
### 关联创建
```go
type CreditCard struct {
  gorm.Model
  Number   string
  UserID   uint
}

type User struct {
  gorm.Model
  Name       string
  CreditCard CreditCard
}

db.Create(&User{
  Name: "jinzhu",
  CreditCard: CreditCard{Number: "411111111111"}
})
// INSERT INTO `users` ...
// INSERT INTO `credit_cards` ...
```
## 查询
### 检索单个对象
```go
// 获取第一条记录（主键升序）
db.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1;

// 获取一条记录，没有指定排序字段
db.Take(&user)
// SELECT * FROM users LIMIT 1;

// 获取最后一条记录（主键降序）
db.Last(&user)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

result := db.First(&user)
result.RowsAffected // 返回找到的记录数
result.Error        // returns error or nil

// 检查 ErrRecordNotFound 错误
errors.Is(result.Error, gorm.ErrRecordNotFound)
```
### 根据主键检索
```go
db.First(&user, 10)
// SELECT * FROM users WHERE id = 10;

db.First(&user, "10")
// SELECT * FROM users WHERE id = 10;

db.Find(&users, []int{1,2,3})
// SELECT * FROM users WHERE id IN (1,2,3);

db.First(&user, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```
### 检索全部对象
```go
// Get all records
result := db.Find(&users)
// SELECT * FROM users;
```
### 条件
#### String条件
```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;

// Get all matched records
db.Where("name <> ?", "jinzhu").Find(&users)
// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name IN ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';

```
#### Struct & Map 条件
```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&users)
// SELECT * FROM users WHERE id IN (20, 21, 22);

// 当使用struct查询时，GORM只对非零字段进行查询，也就是说如果
// 字段的值是0，''，false或其他零值，它将不会被用来建立查询条件
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu";
```
#### 指定结构体查询字段
```go
db.Where(&User{Name: "jinzhu"}, "name", "Age").Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;

db.Where(&User{Name: "jinzhu"}, "Age").Find(&users)
// SELECT * FROM users WHERE age = 0;
```
#### 内联条件
查询条件也可以被内联到 First 和 Find 之类的方法中，其用法类似于 Where。
```go
// Get by primary key if it were a non-integer type
db.First(&user, "id = ?", "string_primary_key")
// SELECT * FROM users WHERE id = 'string_primary_key';

// Plain SQL
db.Find(&user, "name = ?", "jinzhu")
// SELECT * FROM users WHERE name = "jinzhu";

db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Struct
db.Find(&users, User{Age: 20})
// SELECT * FROM users WHERE age = 20;

// Map
db.Find(&users, map[string]interface{}{"age": 20})
// SELECT * FROM users WHERE age = 20;

```
#### Not条件
```go
db.Not("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE NOT name = "jinzhu" ORDER BY id LIMIT 1;

// Not In
db.Not(map[string]interface{}{"name": []string{"jinzhu", "jinzhu 2"}}).Find(&users)
// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Struct
db.Not(User{Name: "jinzhu", Age: 18}).First(&user)
// SELECT * FROM users WHERE name <> "jinzhu" AND age <> 18 ORDER BY id LIMIT 1;

// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
// SELECT * FROM users WHERE id NOT IN (1,2,3) ORDER BY id LIMIT 1;
```
#### Or条件
```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2", Age: 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);

// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2", "age": 18}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);
```
#### 选择特定字段
select 允许选择指定从数据库中检索哪些字段，默认情况下，Gorm 会检索所有字段
```go
db.Select("name", "age").Find(&users)
// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
// SELECT COALESCE(age,'42') FROM users;
```
#### 排序
```go
db.Order("age desc, name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;

// Multiple orders
db.Order("age desc").Order("name").Find(&users)
// SELECT * FROM users ORDER BY age desc, name;

db.Clauses(clause.OrderBy{
  Expression: clause.Expr{SQL: "FIELD(id,?)", Vars: []interface{}{[]int{1, 2, 3}}, WithoutParentheses: true},
}).Find(&User{})
// SELECT * FROM users ORDER BY FIELD(id,1,2,3)
```
#### Limit & Offset
Limit 指定要检索的最大记录数。Offset 指定在开始返回记录前要跳过的记录数
```go
db.Limit(3).Find(&users)
// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
// SELECT * FROM users LIMIT 10; (users1)
// SELECT * FROM users; (users2)

db.Offset(3).Find(&users)
// SELECT * FROM users OFFSET 3;

db.Limit(10).Offset(5).Find(&users)
// SELECT * FROM users OFFSET 5 LIMIT 10;

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
// SELECT * FROM users OFFSET 10; (users1)
// SELECT * FROM users; (users2)
```
#### Group By & Having
```go
type result struct {
  Date  time.Time
  Total int
}

db.Model(&User{}).Select("name, sum(age) as total").Where("name LIKE ?", "group%").Group("name").First(&result)
// SELECT name, sum(age) as total FROM `users` WHERE name LIKE "group%" GROUP BY `name` LIMIT 1


db.Model(&User{}).Select("name, sum(age) as total").Group("name").Having("name = ?", "group").Find(&result)
// SELECT name, sum(age) as total FROM `users` GROUP BY `name` HAVING name = "group"

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Rows()
defer rows.Close()
for rows.Next() {
  ...
}

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Rows()
defer rows.Close()
for rows.Next() {
  ...
}

type Result struct {
  Date  time.Time
  Total int64
}
db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Scan(&results)
```
#### Distinct
从model中选择特定的值
```go
db.Distinct("name", "age").Order("name, age desc").Find(&results)
```
Dinstinct 也可以配合 Pluck，Count 使用
#### Joins
```go
type result struct {
  Name  string
  Email string
}

db.Model(&User{}).Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&result{})
// SELECT users.name, emails.email FROM `users` left join emails on emails.user_id = users.id

rows, err := db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Rows()
for rows.Next() {
  ...
}

db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&results)

// multiple joins with parameter
db.Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "411111111111").Find(&user)

```
#### Joins预加载
```go
db.Joins("Company").Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;

// 带条件的Join
db.Joins("Company", db.Where(&Company{Alive: true})).Find(&users)
// SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id` AND `Company`.`alive` = true;
```
#### Joins一个衍生表
```go
type User struct {
    Id  int
    Age int
}

type Order struct {
    UserId     int
    FinishedAt *time.Time
}

query := db.Table("order").Select("MAX(order.finished_at) as latest").Joins("left join user user on order.user_id = user.id").Where("user.age > ?", 18).Group("order.user_id")
db.Model(&Order{}).Joins("join (?) q on order.finished_at = q.latest", query).Scan(&results)
// SELECT `order`.`user_id`,`order`.`finished_at` FROM `order` join (SELECT MAX(order.finished_at) as latest FROM `order` left join user user on order.user_id = user.id 
// WHERE user.age > 18 GROUP BY `order`.`user_id`) q on order.finished_at = q.latest

```
#### Scan
```go
type Result struct {
  Name string
  Age  int
}

var result Result
db.Table("users").Select("name", "age").Where("name = ?", "Antonio").Scan(&result)

// Raw SQL
db.Raw("SELECT name, age FROM users WHERE name = ?", "Antonio").Scan(&result)
```
## 高级查询
#### 智能选择字段
GORM 允许通过 Select 方法选择特定的字段，如果您在应用程序中经常使用此功能，你也可以定义一个较小的结构体，以实现调用 API 时自动选择特定的字段，例如：
```go
type User struct {
  ID     uint
  Name   string
  Age    int
  Gender string
  // 假设后面还有几百个字段...
}

type APIUser struct {
  ID   uint
  Name string
}

// 查询时会自动选择 `id`、`name` 字段
db.Model(&User{}).Limit(10).Find(&APIUser{})
// SELECT `id`, `name` FROM `users` LIMIT 10
```
#### 子查询
```go
db.Where("amount > (?)", db.Table("orders").Select("AVG(amount)")).Find(&orders)
// SELECT * FROM "orders" WHERE amount > (SELECT AVG(amount) FROM "orders");

subQuery := db.Select("AVG(age)").Where("name LIKE ?", "name%").Table("users")
db.Select("AVG(age) as avgage").Group("name").Having("AVG(age) > (?)", subQuery).Find(&results)
// SELECT AVG(age) as avgage FROM `users` GROUP BY `name` HAVING AVG(age) > (SELECT AVG(age) FROM `users` WHERE name LIKE "name%")

```
#### From子查询
```go
db.Table("(?) as u", db.Model(&User{}).Select("name", "age")).Where("age = ?", 18).Find(&User{})
// SELECT * FROM (SELECT `name`,`age` FROM `users`) as u WHERE `age` = 18

subQuery1 := db.Model(&User{}).Select("name")
subQuery2 := db.Model(&Pet{}).Select("name")
db.Table("(?) as u, (?) as p", subQuery1, subQuery2).Find(&User{})
// SELECT * FROM (SELECT `name` FROM `users`) as u, (SELECT `name` FROM `pets`) as p

```
#### Group条件
```go
db.Where(
    db.Where("pizza = ?", "pepperoni").Where(db.Where("size = ?", "small").Or("size = ?", "medium")),
).Or(
    db.Where("pizza = ?", "hawaiian").Where("size = ?", "xlarge"),
).Find(&Pizza{}).Statement

// SELECT * FROM `pizzas` WHERE (pizza = "pepperoni" AND (size = "small" OR size = "medium")) OR (pizza = "hawaiian" AND size = "xlarge")
```
#### 带多个列的in
```go
db.Where("(name, age, role) IN ?", 
         [][]interface{}{{"jinzhu", 18, "admin"}, 
                         {"jinzhu2", 19, "user"}}).Find(&users)
// SELECT * FROM users WHERE (name, age, role) 
//IN (("jinzhu", 18, "admin"), ("jinzhu 2", 19, "user"));
```
#### 命名参数
```go
db.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
// SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"

db.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu"}).First(&user)
// SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu" ORDER BY `users`.`id` LIMIT 1
```
#### Find至map
```go
result := map[string]interface{}{}
db.Model(&User{}).First(&result, "id = ?", 1)

var results []map[string]interface{}
db.Table("users").Find(&results)
```
#### FirstOrInit
```go
// 未找到 user，则根据给定的条件初始化user
db.FirstOrInit(&user, User{Name: "non_existing"})
// user -> User{Name: "non_existing"}

// 找到了 `name` = `jinzhu` 的 user
db.Where(User{Name: "jinzhu"}).FirstOrInit(&user)
// user -> User{ID: 111, Name: "Jinzhu", Age: 18}

// 找到了 `name` = `jinzhu` 的 user
db.FirstOrInit(&user, map[string]interface{}{"name": "jinzhu"})
// user -> User{ID: 111, Name: "Jinzhu", Age: 18}

// 如果没有找到记录，可以使用包含更多的属性的结构体初始化 user，Attrs 不会被用于生成查询 SQL
// 未找到 user，则根据给定的条件以及 Attrs 初始化 user
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = 'non_existing' ORDER BY id LIMIT 1;
// user -> User{Name: "non_existing", Age: 20}

// 未找到 user，则根据给定的条件以及 Attrs 初始化 user
db.Where(User{Name: "non_existing"}).Attrs("age", 20).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = 'non_existing' ORDER BY id LIMIT 1;
// user -> User{Name: "non_existing", Age: 20}

// 找到了 `name` = `jinzhu` 的 user，则忽略 Attrs
db.Where(User{Name: "Jinzhu"}).Attrs(User{Age: 20}).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = jinzhu' ORDER BY id LIMIT 1;
// user -> User{ID: 111, Name: "Jinzhu", Age: 18}
```
#### FirstOrCreate
获取匹配的第一条记录或者根据给定条件创建一条新纪录（仅 struct, map 条件有效），RowsAffected 返回创建、更新的记录数
```go
// 未找到 User，根据给定条件创建一条新纪录
result := db.FirstOrCreate(&user, User{Name: "non_existing"})
// INSERT INTO "users" (name) VALUES ("non_existing");
// user -> User{ID: 112, Name: "non_existing"}
// result.RowsAffected // => 1

// 找到 `name` = `jinzhu` 的 User
result := db.Where(User{Name: "jinzhu"}).FirstOrCreate(&user)
// user -> User{ID: 111, Name: "jinzhu", "Age": 18}
// result.RowsAffected // => 0

// 如果没有找到记录，可以使用包含更多的属性的结构体创建记录，Attrs 不会被用于生成查询 SQL 。
// 未找到 user，根据条件和 Assign 属性创建记录
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrCreate(&user)
// SELECT * FROM users WHERE name = 'non_existing' ORDER BY id LIMIT 1;
// INSERT INTO "users" (name, age) VALUES ("non_existing", 20);
// user -> User{ID: 112, Name: "non_existing", Age: 20}

// 找到了 `name` = `jinzhu` 的 user，则忽略 Attrs
db.Where(User{Name: "jinzhu"}).Attrs(User{Age: 20}).FirstOrCreate(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;
// user -> User{ID: 111, Name: "jinzhu", Age: 18}

// 不管是否找到记录，Assign 都会将属性赋值给 struct，并将结果写回数据库
// 未找到 user，根据条件和 Assign 属性创建记录
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrCreate(&user)
// SELECT * FROM users WHERE name = 'non_existing' ORDER BY id LIMIT 1;
// INSERT INTO "users" (name, age) VALUES ("non_existing", 20);
// user -> User{ID: 112, Name: "non_existing", Age: 20}

// 找到了 `name` = `jinzhu` 的 user，依然会根据 Assign 更新记录
db.Where(User{Name: "jinzhu"}).Assign(User{Age: 20}).FirstOrCreate(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;
// UPDATE users SET age=20 WHERE id = 111;
// user -> User{ID: 111, Name: "jinzhu", Age: 20}
```
#### 迭代
```go
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Rows()
defer rows.Close()

for rows.Next() {
  var user User
  // ScanRows 方法用于将一行记录扫描至结构体
  db.ScanRows(rows, &user)

  // 业务逻辑...
}
```
#### FindInBatches
```go
// 每次批量处理 100 条
result := db.Where("processed = ?", false).FindInBatches(&results, 100, func(tx *gorm.DB, batch int) error {
  for _, result := range results {
    // 批量处理找到的记录
  }

  tx.Save(&results)

  tx.RowsAffected // 本次批量操作影响的记录数

  batch // Batch 1, 2, 3

  // 如果返回错误会终止后续批量操作
  return nil
})

result.Error // returned error
result.RowsAffected // 整个批量操作影响的记录数
```
#### 查询钩子
```go
func (u *User) AfterFind(tx *gorm.DB) (err error) {
  if u.Role == "" {
    u.Role = "user"
  }
  return
}
```
#### Pluck
```go
var ages []int64
db.Model(&users).Pluck("age", &ages)

var names []string
db.Model(&User{}).Pluck("name", &names)

db.Table("deleted_users").Pluck("name", &names)

// Distinct Pluck
db.Model(&User{}).Distinct().Pluck("Name", &names)
// SELECT DISTINCT `name` FROM `users`

// 超过一列的查询，应该使用 `Scan` 或者 `Find`，例如：
db.Select("name", "age").Scan(&users)
db.Select("name", "age").Find(&users)

```
#### Scope
```go
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
  return db.Where("amount > ?", 1000)
}

func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
  return db.Where("pay_mode_sign = ?", "C")
}

func PaidWithCod(db *gorm.DB) *gorm.DB {
  return db.Where("pay_mode_sign = ?", "C")
}

func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    return db.Where("status IN (?)", status)
  }
}

db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
// 查找所有金额大于 1000 的信用卡订单

db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
// 查找所有金额大于 1000 的货到付款订单

db.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// 查找所有金额大于 1000 且已付款或已发货的订单
```
#### Count
```go
var count int64
db.Model(&User{}).Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Count(&count)
// SELECT count(1) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'

db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)
// SELECT count(1) FROM users WHERE name = 'jinzhu'; (count)

db.Table("deleted_users").Count(&count)
// SELECT count(1) FROM deleted_users;

// Count with Distinct
db.Model(&User{}).Distinct("name").Count(&count)
// SELECT COUNT(DISTINCT(`name`)) FROM `users`

db.Table("deleted_users").Select("count(distinct(name))").Count(&count)
// SELECT count(distinct(name)) FROM deleted_users

// Count with Group
users := []User{
  {Name: "name1"},
  {Name: "name2"},
  {Name: "name3"},
  {Name: "name3"},
}

db.Model(&User{}).Group("name").Count(&count)
count // => 3
```
## 删除
### 软删除
如果模型包含了一个 gorm.deletedat 字段（gorm.Model 已经包含了该字段)，它将自动获得软删除的能力！拥有软删除能力的模型调用 Delete 时，记录不会从数据库中被真正删除。但 GORM 会将 DeletedAt 置为当前时间， 并且不能再通过普通的查询方法找到该记录。
```go
// user 的 ID 是 `111`
db.Delete(&user)
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// 批量删除
db.Where("age = ?", 20).Delete(&User{})
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// 在查询时会忽略被软删除的记录
db.Where("age = 20").Find(&user)
// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
```
### 不引入gorm.Model
```go
type User struct {
  ID      int
  Deleted gorm.DeletedAt
  Name    string
}
```
### 查找被软删除的记录（使用Unscoped）
```go
db.Unscoped().Where("age = 20").Find(&users)
// SELECT * FROM users WHERE age = 20;
```
### 永久删除
```go
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id=10;
```
## 特性解释
### 事务
#### 禁用默认事务
为了确保数据一致性，GORM 会在事务里执行写入操作（创建、更新、删除）。如果没有这方面的要求，您可以在初始化时禁用它，这将获得大约 30%+ 性能提升。
```go
// 全局禁用
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})

// 持续会话模式
tx := db.Session(&Session{SkipDefaultTransaction: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)

```
#### 事务
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

  // 返回 nil 提交事务
  return nil
})
```
#### 嵌套事务
```go
db.Transaction(func(tx *gorm.DB) error {
  tx.Create(&user1)

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user2)
    return errors.New("rollback user2") // Rollback user2
  })

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user3)
    return nil
  })

  return nil
})

// Commit user1, user3

```
#### Save Point、Rollback To Save Point
```go
tx := db.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // Rollback user2

tx.Commit() // Commit user1
```
### Context
GORM 通过 WithContext 方法提供了 Context 支持
#### 单会话模式
| ```go
db.WithContext(ctx).Find(&users)
```
 |
| --- |

#### 持续会话模式
```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```
#### Context超时
对于长 Sql 查询，你可以传入一个带超时的 context 给 db.WithContext 来设置超时时间，例如：
```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

db.WithContext(ctx).Find(&users)
```
#### Hooks/Callbacks 中的 Context
可以从当前 Statement中访问 Context 对象，例如︰
```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    ctx := tx.Statement.Context
    // ...
    return
}
```
#### Chi 中间件示例
在处理 API 请求时持续会话模式会比较有用。例如，可以在中间件中为 *gorm.DB 设置超时 Context，然后使用 *gorm.DB 处理所有请求
Chi 中间件的示例：
```go
func SetDBMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      timeoutContext, _ := context.WithTimeout(context.Background(), time.Second)
      ctx := context.WithValue(r.Context(), "DB", db.WithContext(timeoutContext))
      next.ServeHTTP(w, r.WithContext(ctx))
  })
}

r := chi.NewRouter()
r.Use(SetDBMiddleware)

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
    db, ok := ctx.Value("DB").(*gorm.DB)

    var users []User
    db.Find(&users)

    // 你的其他 DB 操作...
})

r.Get("/user", func(w http.ResponseWriter, r *http.Request) {
    db, ok := ctx.Value("DB").(*gorm.DB)

    var user User
    db.First(&user)

    // 你的其他 DB 操作...
})

```
### DryRun
生成 SQL 但不执行。 它可以用于准备或测试生成的 SQL
```go
// 新建会话模式
stmt := db.Session(&Session{DryRun: true}).First(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 ORDER BY `id`
stmt.Vars         //=> []interface{}{1}

// 全局 DryRun 模式
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{DryRun: true})

// 不同的数据库生成不同的 SQL
stmt := db.Find(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 // PostgreSQL
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = ?  // MySQL
stmt.Vars         //=> []interface{}{1}
```
可以使用下面的代码生成最终的 SQL：
```go
// 注意：SQL 并不总是能安全地执行，GORM 仅将其用于日志，它可能导致会 SQL 注入
db.Dialector.Explain(stmt.SQL.String(), stmt.Vars...)
// SELECT * FROM `users` WHERE `id` = 1
```
### 预编译
PreparedStmt 在执行任何 SQL 时都会创建一个 prepared statement 并将其缓存，以提高后续的效率，例如：
```go
// 全局模式，所有 DB 操作都会创建并缓存预编译语句
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  PrepareStmt: true,
})

// 会话模式
tx := db.Session(&Session{PrepareStmt: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)

// returns prepared statements manager
stmtManger, ok := tx.ConnPool.(*PreparedStmtDB)

// 关闭 *当前会话* 的预编译模式
stmtManger.Close()

// 为 *当前会话* 预编译 SQL
stmtManger.PreparedSQL // => []string{}

// 为当前数据库连接池的（所有会话）开启预编译模式
stmtManger.Stmts // map[string]*sql.Stmt

for sql, stmt := range stmtManger.Stmts {
  sql  // 预编译 SQL
  stmt // 预编译模式
  stmt.Close() // 关闭预编译模式
}
```
### Upsert及冲突
更新数据或插入数据。Upsert是Insert与Update的结合体，表示行存在时执行Update，不存在时执行Insert。执行Upsert时必须要指定完全的Primary key的相关列信息。Upsert语法支持带时间戳的插入和批量写入。
```go
import "gorm.io/gorm/clause"

// 有冲突时什么都不做
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// 当 `id` 有冲突时，更新指定列为默认值
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET ***; SQL Server
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE ***; MySQL

// 当 `id` 有冲突时，更新指定列为新值
db.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET "name"="excluded"."name"; SQL Server
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age"; PostgreSQL
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age=VALUES(age); MySQL

// Update all columns expects primary keys to new value on conflict
db.Clauses(clause.OnConflict{
  UpdateAll: true
}).Create(&users)
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age", ...;
```
### Prometheus
GORM 提供了 Prometheus 插件来收集 [DBStats](https://pkg.go.dev/database/sql?tab=doc#DBStats) 和用户自定义指标
[https://github.com/go-gorm/prometheus](https://github.com/go-gorm/prometheus)
#### 用法
```go
import (
  "gorm.io/gorm"
  "gorm.io/driver/sqlite"
  "gorm.io/plugin/prometheus"
)

db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{})

db.Use(prometheus.New(prometheus.Config{
  DBName:          "db1", // 使用 `DBName` 作为指标 label
  RefreshInterval: 15,    // 指标刷新频率（默认为 15 秒）
  PushAddr:        "prometheus pusher address", // 如果配置了 `PushAddr`，则推送指标
  StartServer:     true,  // 启用一个 http 服务来暴露指标
  HTTPServerPort:  8080,  // 配置 http 服务监听端口，默认端口为 8080 （如果您配置了多个，只有第一个 `HTTPServerPort` 会被使用）
  MetricsCollector: []prometheus.MetricsCollector {
    &prometheus.MySQL{
      VariableNames: []string{"Threads_running"},
    },
  },  // 用户自定义指标
}))
```
#### 用户自定义指标
```go
type MetricsCollector interface {
  Metrics(*Prometheus) []prometheus.Collector
}
```
#### MySQL
GORM 提供了一个示例，说明如何收集 MySQL 状态指标，查看 [prometheus.MySQL](https://github.com/go-gorm/prometheus/blob/master/mysql.go) 获取详情
```go
&prometheus.MySQL{
  // 指标名前缀，默认为 `gorm_status_`
  // 例如： Threads_running 的指标名就是 `gorm_status_Threads_running`
  Prefix: "gorm_status_",
  // 拉取频率，默认使用 Prometheus 的 RefreshInterval
  Interval: 100,
  // 从 SHOW STATUS 选择变量变量，如果不设置，则使用全部的状态变量
  VariableNames: []string{"Threads_running"},
}
```
### Hints
GORM 提供了 optimizer/index/comment hint 支持
[https://github.com/go-gorm/hints](https://github.com/go-gorm/hints)
#### Optimizer Hints
```go
import "gorm.io/hints"

db.Clauses(hints.New("hint")).Find(&User{})
// SELECT * /*+ hint */ FROM `users`
```
#### Index Hints
```go
import "gorm.io/hints"

db.Clauses(hints.UseIndex("idx_user_name")).Find(&User{})
// SELECT * FROM `users` USE INDEX (`idx_user_name`)

db.Clauses(hints.ForceIndex("idx_user_name", "idx_user_id").ForJoin()).Find(&User{})
// SELECT * FROM `users` FORCE INDEX FOR JOIN (`idx_user_name`,`idx_user_id`)"

db.Clauses(
    hints.ForceIndex("idx_user_name", "idx_user_id").ForOrderBy(),
    hints.IgnoreIndex("idx_user_name").ForGroupBy(),
).Find(&User{})
// SELECT * FROM `users` FORCE INDEX FOR ORDER BY (`idx_user_name`,`idx_user_id`) IGNORE INDEX FOR GROUP BY (`idx_user_name`)"

```
#### Comment Hints
```go
import "gorm.io/hints"

db.Clauses(hints.Comment("select", "master")).Find(&User{})
// SELECT /*master*/ * FROM `users`;

db.Clauses(hints.CommentBefore("insert", "node2")).Create(&user)
// /*node2*/ INSERT INTO `users` ...;

db.Clauses(hints.CommentAfter("select", "node2")).Create(&user)
// /*node2*/ INSERT INTO `users` ...;

db.Clauses(hints.CommentAfter("where", "hint")).Find(&User{}, "id = ?", 1)
// SELECT * FROM `users` WHERE id = ? /* hint */
```
## DBResolver
DBResolver 为 GORM 提供了多个数据库支持，支持以下功能：

- 支持多个 sources、replicas
- 读写分离
- 根据工作表、struct 自动切换连接
- 手动切换连接
- Sources/Replicas 负载均衡
- 适用于原生 SQL
- 事务

[https://github.com/go-gorm/dbresolver](https://github.com/go-gorm/dbresolver)
### 用法
```go
import (
  "gorm.io/gorm"
  "gorm.io/plugin/dbresolver"
  "gorm.io/driver/mysql"
)

db, err := gorm.Open(mysql.Open("db1_dsn"), &gorm.Config{})

db.Use(dbresolver.Register(dbresolver.Config{
  // `db2` 作为 sources，`db3`、`db4` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db2_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db3_dsn"), mysql.Open("db4_dsn")},
  // sources/replicas 负载均衡策略
  Policy: dbresolver.RandomPolicy{},
}).Register(dbresolver.Config{
  // `db1` 作为 sources（DB 的默认连接），对于 `User`、`Address` 使用 `db5` 作为 replicas
  Replicas: []gorm.Dialector{mysql.Open("db5_dsn")},
}, &User{}, &Address{}).Register(dbresolver.Config{
  // `db6`、`db7` 作为 sources，对于 `orders`、`Product` 使用 `db8` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db6_dsn"), mysql.Open("db7_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db8_dsn")},
}, "orders", &Product{}, "secondary"))
```
### 自动切换连接
DBResolver 会根据工作表、struct 自动切换连接，对于原生 SQL，DBResolver 会从 SQL 中提取表名以匹配 Resolver，除非 SQL 开头为 SELECT（select for update 除外），否则 DBResolver 总是会使用 sources ，例如：
```go
// `User` Resolver 示例
db.Table("users").Rows() // replicas `db5`
db.Model(&User{}).Find(&AdvancedUser{}) // replicas `db5`
db.Exec("update users set name = ?", "jinzhu") // sources `db1`
db.Raw("select name from users").Row().Scan(&name) // replicas `db5`
db.Create(&user) // sources `db1`
db.Delete(&User{}, "name = ?", "jinzhu") // sources `db1`
db.Table("users").Update("name", "jinzhu") // sources `db1`

// Global Resolver 示例
db.Find(&Pet{}) // replicas `db3`/`db4`
db.Save(&Pet{}) // sources `db2`

// Orders Resolver 示例
db.Find(&Order{}) // replicas `db8`
db.Table("orders").Find(&Report{}) // replicas `db8`

```
### 读写分离
DBResolver 的读写分离目前是基于 [GORM callback](https://gorm.io/docs/write_plugins.html) 实现的。
对于 Query、Row callback，如果手动指定为 Write 模式，此时会使用 sources，否则使用 replicas。 对于 Raw callback，如果 SQL 是以 SELECT 开头，语句会被认为是只读的，会使用 replicas，否则会使用 sources。
### 手动切换连接
```go
// 使用 Write 模式：从 sources db `db1` 读取 user
db.Clauses(dbresolver.Write).First(&user)

// 指定 Resolver：从 `secondary` 的 replicas db `db8` 读取 user
db.Clauses(dbresolver.Use("secondary")).First(&user)

// 指定 Resolver 和 Write 模式：从 `secondary` 的 sources db `db6` 或 `db7` 读取 user
db.Clauses(dbresolver.Use("secondary"), dbresolver.Write).First(&user)
```
### 事务
使用事务时，DBResolver 也会保持使用一个事务，且不会根据配置切换 sources/replicas 连接，但可以在事务开始之前指定使用哪个数据库，例如：
```go
// 通过默认 replicas db 开始事务
tx := DB.Clauses(dbresolver.Read).Begin()

// 通过默认 sources db 开始事务
tx := DB.Clauses(dbresolver.Write).Begin()

// 通过 `secondary` 的 sources db 开始事务
tx := DB.Clauses(dbresolver.Use("secondary"), dbresolver.Write).Begin()

```
### 负载均衡
GORM 支持基于策略的 sources/replicas 负载均衡，自定义策略应该是一个实现了以下接口的 struct：
```go
type Policy interface {
    Resolve([]gorm.ConnPool) gorm.ConnPool
}
```
当前只实现了一个 RandomPolicy 策略，如果没有指定其它策略，它就是默认策略。
### 连接池
```go
db.Use(
  dbresolver.Register(dbresolver.Config{ /* xxx */ }).
  SetConnMaxIdleTime(time.Hour).
  SetConnMaxLifetime(24 * time.Hour).
  SetMaxIdleConns(100).
  SetMaxOpenConns(200)
)
```
## Sharding
Sharding 是一个高性能的 Gorm 分表中间件。它基于 Conn 层做 SQL 拦截、AST 解析、分表路由、自增主键填充，带来的额外开销极小。对开发者友好、透明，使用上与普通 SQL、Gorm 查询无差别，只需要额外注意一下分表键条件。 为您提供高性能的数据库访问。
[https://github.com/go-gorm/sharding](https://github.com/go-gorm/sharding)
### 功能特点

- 非侵入式设计， 加载插件，指定配置，既可实现分表。
- 轻快， 非基于网络层的中间件，像 Go 一样快支持多种数据库。
- PostgreSQL 已通过测试，MySQL 和 SQLite 也在路上。
- 多种主键生成方式支持（Snowflake, PostgreSQL Sequence, 以及自定义支持）Snowflake 支持从主键中确定分表键。
### 使用说明
配置 Sharding 中间件，为需要分表的业务表定义他们分表的规则。 查看 [Godoc](https://pkg.go.dev/github.com/go-gorm/sharding) 获取配置详情。
```go
import (
    "fmt"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/sharding"
)

dsn := "postgres://localhost:5432/sharding-db?sslmode=disable"
db, err := gorm.Open(postgres.New(postgres.Config{DSN: dsn}))

db.Use(sharding.Register(sharding.Config{
    ShardingKey:         "user_id",
    NumberOfShards:      64,
    PrimaryKeyGenerator: sharding.PKSnowflake,
}, "orders").Register(sharding.Config{
    ShardingKey:         "user_id",
    NumberOfShards:      256,
    PrimaryKeyGenerator: sharding.PKSnowflake,
    // This case for show up give notifications, audit_logs table use same sharding rule.
}, Notification{}, AuditLog{}))
```
依然保持原来的方式使用 db 来查询数据库。 你只需要注意在 CURD 动作的时候，明确知道 Sharding Key 对应的分表，查询条件带 Sharding Key，以确保 Sharding 能理解数据需要对应到哪一个子表。
```go
// GORM 创建示例，这会插入到 orders_02 表
db.Create(&Order{UserID: 2})
// sql: INSERT INTO orders_2 ...

// 原生 SQL 插入示例，这会插入到 orders_03 表
db.Exec("INSERT INTO orders(user_id) VALUES(?)", int64(3))

// 这会抛出 ErrMissingShardingKey 错误，因此此处没有提供 sharding key
db.Create(&Order{Amount: 10, ProductID: 100})
fmt.Println(err)

// Find 方法，这会检索 order_02 表
var orders []Order
db.Model(&Order{}).Where("user_id", int64(2)).Find(&orders)
fmt.Printf("%#v\n", orders)

// 原生 SQL 也是支持的
db.Raw("SELECT * FROM orders WHERE user_id = ?", int64(3)).Scan(&orders)
fmt.Printf("%#v\n", orders)

// 这会抛出 ErrMissingShardingKey 错误，因为 WHERE 条件没有包含 sharding key
err = db.Model(&Order{}).Where("product_id", "1").Find(&orders).Error
fmt.Println(err)

// Update 和 Delete 方法与创建、查询类似
db.Exec("UPDATE orders SET product_id = ? WHERE user_id = ?", 2, int64(3))
err = db.Exec("DELETE FROM orders WHERE product_id = 3").Error
fmt.Println(err) // ErrMissingShardingKey
```
## 索引
### 索引标签
Gorm可以接受很多索引设置，例如
> class、type、where、comment、expression、sort、collate、option

```go
type User struct {
    Name  string `gorm:"index"`
    Name2 string `gorm:"index:idx_name,unique"`
    Name3 string `gorm:"index:,sort:desc,collate:utf8,type:btree,length:10,where:name3 != 'jinzhu'"`
    Name4 string `gorm:"uniqueIndex"`
    Age   int64  `gorm:"index:,class:FULLTEXT,comment:hello \\, world,where:age > 10"`
    Age2  int64  `gorm:"index:,expression:ABS(age)"`
}

// MySQL 选项
type User struct {
    Name string `gorm:"index:,class:FULLTEXT,option:WITH PARSER ngram INVISIBLE"`
}

// PostgreSQL 选项
type User struct {
    Name string `gorm:"index:,option:CONCURRENTLY"`
}
```
### 唯一索引
uniqueIndex 标签的作用与 index 类似，等效于 index: ，unique
```go
type User struct {
    Name1 string `gorm:"uniqueIndex"`
    Name2 string `gorm:"uniqueIndex:idx_name,sort:desc"`
}
```
### 复合索引
两个字段使用同一个索引名将创建复合索引，例如：
```go
// create composite index `idx_member` with columns `name`, `number`
type User struct {
    Name   string `gorm:"index:idx_member"`
    Number string `gorm:"index:idx_member"`
}
```
#### 字段优先级
复合索引列的顺序会影响其性能，因此必须仔细考虑
可以使用 priority 指定顺序，默认优先级值是 10，如果优先级值相同，则顺序取决于模型结构体字段的顺序
```go
type User struct {
    Name   string `gorm:"index:idx_member"`
    Number string `gorm:"index:idx_member"`
}
// column order: name, number

type User struct {
    Name   string `gorm:"index:idx_member,priority:2"`
    Number string `gorm:"index:idx_member,priority:1"`
}
// column order: number, name

type User struct {
    Name   string `gorm:"index:idx_member,priority:12"`
    Number string `gorm:"index:idx_member"`
}
// column order: number, name
```
#### 共享复合索引
如果正在创建带有嵌入结构的共享复合索引，您不能指定索引名称。 作为嵌入结构不止一次会导致重复的索引名称在 db。
在这种情况下，可以使用索引标签 composite, 它意味着复合索引的id。 所有具有相同结构复合id的字段都与原始规则一样，被合并到相同的索引中。 但改进使得最受影响/嵌入结构能够通过命名策略生成索引名称。 例如:
```go
type Foo struct {
  IndexA int `gorm:"index:,unique,composite:myname"`
  IndexB int `gorm:"index:,unique,composite:myname"`
}
```
如果表Foo被创建，复合索引的名称将是 idx_foo_myname。
```go
type Bar0 struct {
  Foo
}

type Bar1 struct {
  Foo
}
```
复合索引的名称分别是 idx_bar0_myname 和 idx_bar1_myname。
复合 只能在指定索引名称时使用。
### 多索引
一个字段接受多个 index、uniqueIndex 标签，这会在一个字段上创建多个索引
```go
type UserIndex struct {
    OID          int64  `gorm:"index:idx_id;index:idx_oid,unique"`
    MemberNumber string `gorm:"index:idx_id"`
}
```
## Scopes
作用域允许复用通用的逻辑，这种共享逻辑需要定义为类型 func(*gorm.DB) *gorm.DB
### 查询
Scope查询示例
```go
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
    return db.Where("amount > ?", 1000)
}

func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func PaidWithCod(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
    return func (db *gorm.DB) *gorm.DB {
      return db.Where("status IN (?)", status)
  }
}

db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
// 查找所有金额大于 1000 的信用卡订单

db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
// 查找所有金额大于 1000 的 COD 订单

db.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// 查找所有金额大于1000 的已付款或已发货订单
```
### 分页
```go
func Paginate(r *http.Request) func(db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    q := r.URL.Query()
    page, _ := strconv.Atoi(q.Get("page"))
    if page == 0 {
      page = 1
    }

    pageSize, _ := strconv.Atoi(q.Get("page_size"))
    switch {
    case pageSize > 100:
      pageSize = 100
    case pageSize <= 0:
      pageSize = 10
    }

    offset := (page - 1) * pageSize
    return db.Offset(offset).Limit(pageSize)
  }
}

db.Scopes(Paginate(r)).Find(&users)
db.Scopes(Paginate(r)).Find(&articles)

```
### 动态表
使用 Scopes 来动态指定查询的表
```go
func TableOfYear(user *User, year int) func(db *gorm.DB) *gorm.DB {
  return func(db *gorm.DB) *gorm.DB {
        tableName := user.TableName() + strconv.Itoa(year)
        return db.Table(tableName)
  }
}

DB.Scopes(TableOfYear(user, 2019)).Find(&users)
// SELECT * FROM users_2019;

DB.Scopes(TableOfYear(user, 2020)).Find(&users)
// SELECT * FROM users_2020;

// Table form different database
func TableOfOrg(user *User, dbName string) func(db *gorm.DB) *gorm.DB {
  return func(db *gorm.DB) *gorm.DB {
        tableName := dbName + "." + user.TableName()
        return db.Table(tableName)
  }
}

DB.Scopes(TableOfOrg(user, "org1")).Find(&users)
// SELECT * FROM org1.users;

DB.Scopes(TableOfOrg(user, "org2")).Find(&users)
// SELECT * FROM org2.users;

```
### 更新
Scope 更新、删除示例
```go
func CurOrganization(r *http.Request) func(db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    org := r.Query("org")

    if org != "" {
      var organization Organization
      if db.Session(&Session{}).First(&organization, "name = ?", org).Error == nil {
        return db.Where("org_id = ?", organization.ID)
      }
    }

    db.AddError("invalid organization")
    return db
  }
}

db.Model(&article).Scopes(CurOrganization(r)).Update("Name", "name 1")
// UPDATE articles SET name = "name 1" WHERE org_id = 111
db.Scopes(CurOrganization(r)).Delete(&Article{})
// DELETE FROM articles WHERE org_id = 111

```
## 性能
### 禁用默认事务
对于写操作（创建、更新、删除），为了确保数据的完整性，GORM会将它们封装在一个事务里，但是会降低性能，可以在初始化时禁用这种方式：
```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})
```
### 缓存预编译语句
执行任何SQL时都创建并缓存预编译语句，可以提高后续的调用速度
```go
// 全局模式
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  PrepareStmt: true,
})

// 会话模式
tx := db.Session(&Session{PrepareStmt: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)
```
### 带 PreparedStmt 的 SQL 预处理语句
#### 好处

- 查询只需要被解析（或准备）一次，但可以使用相同或不同的参数执行多次。当查询准备好（Prepared）之后，数据库就会分析，编译并优化它要执行查询的计划。对于复杂查询来说，如果你要重复执行许多次有不同参数的但结构相同的查询，这个过程会占用大量的时间，使得你的应用变慢。通过使用一个预处理语句你就可以避免重复分析、编译、优化的环节。简单来说，预处理语句使用更少的资源，执行速度也就更快。
- 传给预处理语句的参数不需要使用引号，底层驱动会为你处理这个。如果你的应用独占地使用预处理语句，你就可以确信没有SQL注入会发生。（然而，如果你仍然在用基于不受信任的输入来构建查询的其他部分，这仍然是具有风险的）。

Prepared Statement 也可以和原生 SQL 一起使用，例如：
```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  PrepareStmt: true,
})

db.Raw("select sum(age) from users where role = ?", "admin").Scan(&age)
```


