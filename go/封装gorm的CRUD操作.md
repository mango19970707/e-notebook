### 封装gorm的CRUD操作

---

#### 1、新增
```go
// 新增一条数据
func CreateEntity(DBExecutor *gorm.DB, entity interface{}) error {
    if err := DBExecutor.Debug().Create(entity).Error; err != nil {
        return err
    }
    return nil
}

//    新增多条数据
func CreateEntities(DBExecutor *gorm.DB, entities interface{}) error {
DBResult := DBExecutor.Create(entities)
if err := DBResult.Error; err != nil {
return err
}
return nil
}
```

#### 2、删除
```go
//  删除一条数据
func DeleteEntity(DBExecutor *gorm.DB, entity interface{}) (int64, error) {
    DBResult := DBExecutor.Unscoped().Where(entity).Delete(entity)
    if err := DBResult.Error; err != nil {
        return 0, err
    }
    return DBResult.RowsAffected, nil
}

//  删除多条数据
func DeleteEntities(DBExecutor *gorm.DB, filter *map[string]interface{}, entity interface{}) (int64, error) {
query := DBExecutor.Unscoped()
if filter != nil {
for key, value := range *filter {
switch {
case strings.HasSuffix(key, "IN"):
query = query.Where(key+" (?)", value)
default:
query = query.Where(key+" = ?", value)
}
}
}
result := query.Debug().Delete(entity)
return result.RowsAffected, result.Error
}
```

#### 3、查询
```go
// 通过ID查询
func QueryEntity(entityID interface{}, entity interface{}) error {
    result := DB.Where("ID = ?", entityID).First(entity)
    if result.RowsAffected < 1 {
        return errors.New("查询结果为空")
    }
    return result.Error
}

// 带有group、order、where筛选条件的查询,不存在时，不会返回错误
func QueryEntityByFilter(filter *map[string]interface{}, entity interface{}) error {
    query := DB.Select("*")
    if filter != nil {
        for key, value := range *filter {
            if key == "order" {
            query = query.Order(value)
            } else if key == "group" {
            query = query.Group(value.(string))
            } else {
            query = query.Where(key+" = ?", value)
            }
        }
    }
    result := query.Find(entity)
    return result.Error
}

//    查询数目
func QueryCount(params *map[string]interface{}, list interface{}, count *int64) error {
    query := DB
    if params != nil {
        for key, value := range *params {
            switch {
            case strings.HasSuffix(key, "IN"):
                query = query.Where(key+" (?)", value)
            case key == "distinct":
                query = query.Distinct(value)
            case strings.HasSuffix(key, "!="):
            if value == "" {
                query = query.Where(key + " ''")
            } else if value == 0 {
                query = query.Where(key + " 0")
            } else {
                query = query.Where(key, value)
            }
            default:
                query = query.Where(key+" = ?", value)
            }
        }
    }
    if err := query.Find(list).Count(count).Error; err != nil {
        return err
    }
    return nil
}

// 带筛选条件查询
func QueryList(params *map[string]interface{}, list interface{}) error {
query := DB.Debug()
if params != nil {
for key, value := range *params {
switch {
case key == "select":
valueStr := ""
flagStr := ""
for _, v := range value.([]string) {
valueStr = valueStr + flagStr + v
flagStr = " "
}
query = query.Select(valueStr)
case key == "table":
query = query.Table(value.(string))
case key == "distinct":
query = query.Distinct(value)
case key == "order":
query = query.Order(value)
case key == "limit":
query = query.Limit(value.(int))
case key == "offset":
query = query.Offset(value.(int))
case strings.HasSuffix(key, "IN"):
query = query.Where(key+" (?)", value)
case strings.HasSuffix(key, "!="):
if value == "" {
query = query.Where(key + " ''")
} else if value == 0 {
query = query.Where(key + " 0")
} else {
query = query.Where(key, value)
}
case strings.HasSuffix(key, "?"):
query = query.Where(key, value)
case strings.HasSuffix(key, "LIKE"):
query = query.Where(key+" ?", "%"+value.(string)+"%")
case strings.Index(key, "LIKE") != -1:
queryInfo := strings.SplitN(value.(string), " ", 2)
query = query.Where(queryInfo[0]+" LIKE ?", "%"+queryInfo[1]+"%")
case strings.HasSuffix(key, "BETWEEN"):
values := strings.Split(value.(string), ",")
query = query.Where(key+"? AND ?", values[0], values[1])
default:
query = query.Where(key+" = ?", value)
}
}
}
if err := query.Find(list).Error; err != nil {
return err
}
return nil
}

//    查询并返回查询到的结果的总数
func QueryListReturnNumber(params *map[string]interface{}, list interface{}) (int64, error) {
var total int64
query := DB
if params != nil {
for key, value := range *params {
switch {
case key == "select":
valueStr := ""
flagStr := ""
for _, v := range value.([]string) {
valueStr = valueStr + flagStr + v
flagStr = ","
}
query = query.Select(valueStr)
case key == "distinct":
query = query.Distinct(value)
case key == "order":
query = query.Order(value)
case key == "limit":
continue
// query = query.Limit(value.(int))
case key == "offset":
continue
// query = query.Offset(value.(int))
case key == "time":
query = query.Where("created_at >= ? AND created_at <= ?", value.([]time.Time)[0], value.([]time.Time)[1])
case key == "data_time":
query = query.Where("data_time >= ? AND data_time <= ?", value.([]time.Time)[0], value.([]time.Time)[1])
case strings.Index(key, "LIKE") != -1:
queryInfo := strings.Split(value.(string), " ")
query = query.Where(queryInfo[0]+" LIKE ?", "%"+queryInfo[1]+"%")
case strings.HasSuffix(key, "BETWEEN"):
values := strings.Split(value.(string), ",")
query = query.Where("? between ? AND ?", values[0], values[1], values[2])
case strings.HasSuffix(key, "?"):
query = query.Where(key, value)
case strings.HasSuffix(key, "IN"):
query = query.Where(key+" (?)", value)
default:
query = query.Where(key+" = ?", value)
}
}
// 这里为了获取分页前结果的总个数，给前端计算页数使用
total = query.Find(list).RowsAffected
for key, value := range *params {
if key == "limit" {
query = query.Limit(value.(int))
} else if key == "offset" {
query = query.Offset(value.(int))
} else {
continue
}
}
}
if err := query.Debug().Find(list).Error; err != nil {
return 0, err
}
return total, nil
}

// 关联查询
type result struct {
dal.Comment
User1Name string `json:"user1_name"`
User2Name string `json:"user2_name"`
}
var res []*result
err = dal.DB.Table("comments").Select("comments.*,u1.name as user1_name,u2.name as user2_name").
Joins("left join users as u1 on u1.id = comments.user1_id").
Joins("left join users as u2 on u2.id = comments.user2_id").
Where("comments.user1_id = ? or comments.user2_id = ?",int(m["user_id"].(float64)),int(m["user_id"].(float64))).
Where("comments.status = 2").
Scan(&res).Error
if err!=nil{
log.Printf("get comment error:unit not exist:%v", err)
c.JSON(404, gin.H{
"message":"comment not exist",
})
return
}
```

#### 4、更新
```go
//    更新多条数据
func UpdateEntities(DBExecutor *gorm.DB, entities interface{}) error {
    result := DBExecutor.Save(entities)
    if err := result.Error; err != nil {
        return err
    }
    return nil
}

//    更新指定字段
func UpdateFields(DBExecutor *gorm.DB, model interface{}, selector *map[string]interface{}, fields *map[string]interface{}) error {
query := DBExecutor.Model(&model)
if selector != nil {
for key, value := range *selector {
switch {
case key == "order":
case strings.HasSuffix(key, "IN"):
query = query.Where(key+" (?)", value)
default:
query = query.Where(key+" = ?", value)
}
}
}
result := query.Updates(fields)
if err := result.Error; err != nil {
return err
}
return nil
}

//    更新单条数据
func UpdateEntity(DBExecutor *gorm.DB, entity interface{}) error {
result := DBExecutor.Model(entity).Updates(entity)
if err := result.Error; err != nil {
return err
}
return nil
}

//    根据ID筛选更新数据
func UpdateEntityByID(DBExecutor *gorm.DB, ID uint, entity interface{}) error {
result := DBExecutor.Model(entity).Where("id = ?", ID).Updates(entity)
if err := result.Error; err != nil {
return err
}
return nil
}

//    查询到结果就返回，否则插入新数据
func GetOrCreate(DBExecutor *gorm.DB, entity interface{}) (int64, error) {
// 返回的int, 表示查询到的个数, 0则代表查询无果，创建之
result := DBExecutor.Where(entity).Limit(1).Find(entity)
if err = result.Error; err != nil {
return 0, err
}
if result.RowsAffected == 0 {
err = CreateEntity(DBExecutor, entity)
if err != nil {
return 0, err
}
return 0, nil
}
return result.RowsAffected, nil
}

//    数据已存在就更新，否则创建
func UpdateOrCreate(DBExecutor *gorm.DB, entity interface{}) error {
result := DBExecutor.Where(entity).Limit(1).Find(entity)
if err = result.Error; err != nil {
return err
}
if result.RowsAffected == 0 {
err = CreateEntity(DBExecutor, entity)
if err != nil {
return err
}
} else if result.RowsAffected == 1 {
result = DBExecutor.Model(entity).Updates(entity)
if err := result.Error; err != nil {
return err
}
}
return nil
}
```