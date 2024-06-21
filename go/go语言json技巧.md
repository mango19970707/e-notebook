### go语言json技巧

---

#### 1、忽略某个字段
如果想忽略某个字段，可以按如下方式在tag中添加-。
```go
type Person struct {
    Name   string `json:"name"`
    Age    int64
    Weight float64 `json:"-"` // 指定json序列化/反序列化时忽略此字段
}
```

#### 2、忽略零值字段
当 struct 中的字段没有值时， json.Marshal() 序列化的时候不会忽略这些字段，如果想忽略零值，可以在对应字段添加omitempty 。
```go
type User struct {
    Name  string   `json:"name"`
    Email string   `json:"email,omitempty"`
    Hobby []string `json:"hobby,omitempty"`
}
```

#### 3、序列化嵌套结构体
```go
type Profile struct {
    Website string `json:"site"`
    Slogan  string `json:"slogan"`
}
 
type User struct {
    Name  string   `json:"name"`
    Email string   `json:"email,omitempty"`
    Hobby []string `json:"hobby,omitempty"`
    Profile
}
//序列化结果：str:{"name":"七米","hobby":["足球","双色球"],"site":"","slogan":""}
 
type User struct {
    Name    string   `json:"name"`
    Email   string   `json:"email,omitempty"`
    Hobby   []string `json:"hobby,omitempty"`
    Profile `json:"profile"`
}
// 序列化结果：str:{"name":"七米","hobby":["足球","双色球"],"profile":{"site":"","slogan":""}}
```

#### 4、忽略嵌套结构体的零值
想要在嵌套的结构体为空值时，忽略该字段，仅添加omitempty是不够的，还需要使用嵌套的结构体指针。
```go
type User struct {
    Name     string   `json:"name"`
    Email    string   `json:"email,omitempty"`
    Hobby    []string `json:"hobby,omitempty"`
    *Profile `json:"profile,omitempty"`
}
// str:{"name":"七米","hobby":["足球","双色球"]}
```

#### 5、不修改原结构体忽略空值字段
我们需要json序列化User，但是不想把密码也序列化，又不想修改User结构体，这个时候我们就可以使用创建另外一个结构体PublicUser匿名嵌套原User，同时指定Password字段为匿名结构体指针类型，并添加omitemptytag，示例代码如下：
```go
type User struct {
    Name     string `json:"name"`
    Password string `json:"password"`
}
 
type PublicUser struct {
    *User             // 匿名嵌套
    Password *struct{} `json:"password,omitempty"`
}
 
func omitPasswordDemo() {
    u1 := User{
        Name:     "七米",
        Password: "123456",
    }
    b, err := json.Marshal(PublicUser{User: &u1})
    if err != nil {
        fmt.Printf("json.Marshal u1 failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)  // str:{"name":"七米"}
}
```

#### 6、优雅处理字符串格式的数字
有时候，前端在传递来的json数据中可能会使用字符串类型的数字，这个时候可以在结构体tag中添加string来告诉json包从字符串中解析相应字段的数据。
```go
type Card struct {
    ID    int64   `json:"id,string"`    // 添加string tag
    Score float64 `json:"score,string"` // 添加string tag
}
 
func intAndStringDemo() {
    jsonStr1 := `{"id": "1234567","score": "88.50"}`
    var c1 Card
    if err := json.Unmarshal([]byte(jsonStr1), &c1); err != nil {
        fmt.Printf("json.Unmarsha jsonStr1 failed, err:%v\n", err)
        return
    }
    fmt.Printf("c1:%#v\n", c1) // c1:main.Card{ID:1234567, Score:88.5}
}
```

#### 7、解决反序列化整数变浮点数的问题
因为在 JSON 协议中是没有整型和浮点型之分的，它们统称为number。json字符串中的数字经过Go语言中的json包反序列化之后都会成为float64类型。
```go
// useNumberDemo 使用json.UseNumber
// 解决将JSON数据反序列化成map[string]interface{}时
// 数字变为科学计数法表示的浮点数问题
func useNumberDemo(){
    type student struct {
        ID int64 `json:"id"`
        Name string `json:"q1mi"`
    }
    s := student{ID: 123456789,Name: "q1mi"}
    b, _ := json.Marshal(s)
    var m map[string]interface{}
    // decode
    json.Unmarshal(b, &m)
    fmt.Printf("id:%#v\n", m["id"])  // 1.23456789e+08
    fmt.Printf("id type:%T\n", m["id"])  //float64
 
    // use Number decode
    decoder := json.NewDecoder(bytes.NewReader(b))
    decoder.UseNumber()
    decoder.Decode(&m)
    fmt.Printf("id:%#v\n", m["id"])  // "123456789"
    fmt.Printf("id type:%T\n", m["id"]) // json.Number
}
```

#### 8、自定义解析时间字段
Go语言内置的 json 包使用 RFC3339 标准中定义的时间格式，对我们序列化时间字段的时候有很多限制，例如：
```go
type Post struct {
    CreateTime time.Time `json:"create_time"`
}
 
func timeFieldDemo() {
    p1 := Post{CreateTime: time.Now()}
    b, err := json.Marshal(p1)
    if err != nil {
        fmt.Printf("json.Marshal p1 failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)
    jsonStr := `{"create_time":"2020-04-05 12:25:42"}`
    var p2 Post
    if err := json.Unmarshal([]byte(jsonStr), &p2); err != nil {
        fmt.Printf("json.Unmarshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("p2:%#v\n", p2)
}
 
//报错：json.Unmarshal failed, err:parsing time ""2020-04-05 12:25:42"" as ""2006-01-02T15:04:05Z07:00"": cannot parse " 12:25:42"" as "T"
```
也就是内置的json包不识别我们常用的字符串时间格式，如2020-04-05 12:25:42。
不过我们通过实现 json.Marshaler/json.Unmarshaler 接口实现自定义的事件格式解析。
```go
type CustomTime struct {
    time.Time
}
 
const ctLayout = "2006-01-02 15:04:05"
 
var nilTime = (time.Time{}).UnixNano()
 
func (ct *CustomTime) UnmarshalJSON(b []byte) (err error) {
    s := strings.Trim(string(b), "\"")
    if s == "null" {
        ct.Time = time.Time{}
        return
    }
    ct.Time, err = time.Parse(ctLayout, s)
    return
}
 
func (ct *CustomTime) MarshalJSON() ([]byte, error) {
    if ct.Time.UnixNano() == nilTime {
        return []byte("null"), nil
    }
    return []byte(fmt.Sprintf("\"%s\"", ct.Time.Format(ctLayout))), nil
}
 
func (ct *CustomTime) IsSet() bool {
    return ct.UnixNano() != nilTime
}
 
type Post struct {
    CreateTime CustomTime `json:"create_time"`
}
 
func timeFieldDemo() {
    p1 := Post{CreateTime: CustomTime{time.Now()}}
    b, err := json.Marshal(p1)
    if err != nil {
        fmt.Printf("json.Marshal p1 failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)
    jsonStr := `{"create_time":"2020-04-05 12:25:42"}`
    var p2 Post
    if err := json.Unmarshal([]byte(jsonStr), &p2); err != nil {
        fmt.Printf("json.Unmarshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("p2:%#v\n", p2)
}
```

#### 9、自定义MarshalJSON和UnmarshalJSON方法
如果你能够为某个类型实现了MarshalJSON()([]byte, error)和UnmarshalJSON(b []byte) error方法，那么这个类型在序列化（MarshalJSON）/反序列化（UnmarshalJSON）时就会使用你定制的相应方法。
```go
type Order struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    CreatedTime time.Time `json:"created_time"`
}
 
const layout = "2006-01-02 15:04:05"
 
// MarshalJSON 为Order类型实现自定义的MarshalJSON方法
func (o *Order) MarshalJSON() ([]byte, error) {
    type TempOrder Order // 定义与Order字段一致的新类型
    return json.Marshal(struct {
        CreatedTime string `json:"created_time"`
        *TempOrder         // 避免直接嵌套Order进入死循环
    }{
        CreatedTime: o.CreatedTime.Format(layout),
        TempOrder:   (*TempOrder)(o),
    })
}
 
// UnmarshalJSON 为Order类型实现自定义的UnmarshalJSON方法
func (o *Order) UnmarshalJSON(data []byte) error {
    type TempOrder Order // 定义与Order字段一致的新类型
    ot := struct {
        CreatedTime string `json:"created_time"`
        *TempOrder         // 避免直接嵌套Order进入死循环
    }{
        TempOrder: (*TempOrder)(o),
    }
    if err := json.Unmarshal(data, &ot); err != nil {
        return err
    }
    var err error
    o.CreatedTime, err = time.Parse(layout, ot.CreatedTime)
    if err != nil {
        return err
    }
    return nil
}
 
// 自定义序列化方法
func customMethodDemo() {
    o1 := Order{
        ID:          123456,
        Title:       "《七米的Go学习笔记》",
        CreatedTime: time.Now(),
    }
    // 通过自定义的MarshalJSON方法实现struct -> json string
    b, err := json.Marshal(&o1)
    if err != nil {
        fmt.Printf("json.Marshal o1 failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)
    // 通过自定义的UnmarshalJSON方法实现json string -> struct
    jsonStr := `{"created_time":"2020-04-05 10:18:20","id":123456,"title":"《七米的Go学习笔记》"}`
    var o2 Order
    if err := json.Unmarshal([]byte(jsonStr), &o2); err != nil {
        fmt.Printf("json.Unmarshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("o2:%#v\n", o2)
}
```

#### 10、使用匿名结构体添加字段
使用内嵌结构体能够扩展结构体的字段，但有时候我们没有必要单独定义新的结构体，可以使用匿名结构体简化操作。
```go
type UserInfo struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
 
func anonymousStructDemo() {
    u1 := UserInfo{
        ID:   123456,
        Name: "七米",
    }
    // 使用匿名结构体内嵌User并添加额外字段Token
    b, err := json.Marshal(struct {
        *UserInfo
        Token string `json:"token"`
    }{
        &u1,
        "91je3a4s72d1da96h",
    })
    if err != nil {
        fmt.Printf("json.Marsha failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)
    // str:{"id":123456,"name":"七米","token":"91je3a4s72d1da96h"}
}
```

#### 11、使用匿名结构体组合多个结构体
同理，也可以使用匿名结构体来组合多个结构体来序列化与反序列化数据。
```go
type Comment struct {
    Content string
}
 
type Image struct {
    Title string `json:"title"`
    URL   string `json:"url"`
}
 
func anonymousStructDemo2() {
    c1 := Comment{
        Content: "永远不要高估自己",
    }
    i1 := Image{
        Title: "赞赏码",
        URL:   "https://www.liwenzhou.com/images/zanshang_qr.jpg",
    }
    // struct -> json string
    b, err := json.Marshal(struct {
        *Comment
        *Image
    }{&c1, &i1})
    if err != nil {
        fmt.Printf("json.Marshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("str:%s\n", b)
    // json string -> struct
    jsonStr := `{"Content":"永远不要高估自己","title":"赞赏码","url":"https://www.liwenzhou.com/images/zanshang_qr.jpg"}`
    var (
        c2 Comment
        i2 Image
    )
    if err := json.Unmarshal([]byte(jsonStr), &struct {
        *Comment
        *Image
    }{&c2, &i2}); err != nil {
        fmt.Printf("json.Unmarshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("c2:%#v i2:%#v\n", c2, i2)
}
```

#### 12、处理不确定层级的json
如果json串没有固定的格式导致不好定义与其相对应的结构体时，我们可以使用json.RawMessage原始字节数据保存下来。
```go
type sendMsg struct {
    User string `json:"user"`
    Msg  string `json:"msg"`
}
 
func rawMessageDemo() {
    jsonStr := `{"sendMsg":{"user":"q1mi","msg":"永远不要高估自己"},"say":"Hello"}`
    // 定义一个map，value类型为json.RawMessage，方便后续更灵活地处理
    var data map[string]json.RawMessage
    if err := json.Unmarshal([]byte(jsonStr), &data); err != nil {
        fmt.Printf("json.Unmarshal jsonStr failed, err:%v\n", err)
        return
    }
    var msg sendMsg
    if err := json.Unmarshal(data["sendMsg"], &msg); err != nil {
        fmt.Printf("json.Unmarshal failed, err:%v\n", err)
        return
    }
    fmt.Printf("msg:%#v\n", msg)
    // msg:main.sendMsg{User:"q1mi", Msg:"永远不要高估自己"}
}
```