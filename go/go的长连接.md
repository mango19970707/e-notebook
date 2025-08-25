### go的长连接

---

#### 1、golang的持久化连接
在Golang中使用持久化连接发起HTTP请求，主要依赖Transport，官方封装的net库中已经支持。Transport实现了RoundTripper接口，该接口只有一个方法RoundTrip()，故Transport的入口函数就是RoundTrip()。
Transport的主要功能：
- 缓存了长连接，用于大量http请求场景下的连接复用
- 对连接做一些限制，连接超时时间，每个host的最大连接数

#### 2、代码示例
```go
package main
 
import (
    "fmt"
    "io/ioutil"
    "net"
    "net/http"
    "time"
)
 
var HTTPTransport = &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second, // 连接超时时间
        KeepAlive: 60 * time.Second, // 保持长连接的时间
    }).DialContext, // 设置连接的参数
    MaxIdleConns:          500, // 最大空闲连接
    IdleConnTimeout:       60 * time.Second, // 空闲连接的超时时间
    ExpectContinueTimeout: 30 * time.Second, // 等待服务第一个响应的超时时间
    MaxIdleConnsPerHost:   100, // 每个host保持的空闲连接数
}
 
func main() {
    times := 50
    uri := "http://local.test.com/t.php"
    // uri := "http://www.baidu.com"
     
 
    // 短连接的情况
 
    start := time.Now()
    client := http.Client{} // 初始化http的client
    for i := 0; i < times; i++ {
        req, err := http.NewRequest(http.MethodGet, uri, nil)
        if err != nil {
            panic("Http Req Failed " + err.Error())
        }
        resp, err := client.Do(req) // 发起请求
        if err != nil {
            panic("Http Request Failed " + err.Error())
        }
        defer resp.Body.Close()
        ioutil.ReadAll(resp.Body)
    }
    fmt.Println("Orig GoNet Short Link", time.Since(start))
     
 
    // 长连接的情况
 
    start2 := time.Now()
    client2 := http.Client{Transport: HTTPTransport} // 初始化一个带有transport的http的client
    for i := 0; i < times; i++ {
        req, err := http.NewRequest(http.MethodGet, uri, nil)
        if err != nil {
            panic("Http Req Failed " + err.Error())
        }
        resp, err := client2.Do(req)
        if err != nil {
            panic("Http Request Failed " + err.Error())
        }
        defer resp.Body.Close()
        ioutil.ReadAll(resp.Body) // 如果不及时从请求中获取结果，此连接会占用，其他请求服务复用连接
    }
    fmt.Println("Orig GoNet Long Link", time.Since(start2))
}
```

#### 3、注意事项
如果发起请求后，而不获取请求的结果，即缺少如下代码：
```go
ioutil.ReadAll(resp.Body)
```
则客户端会重新建立连接发起新的请求，也即没有利用到持久化连接的优势，通过netstat可以查看到连接在持续增加。因此，只要请求响应包含响应body必须读取出来，否则无法复用连接。
如果响应数据不需要可以使用下面示例丢弃响应：
```go
io.Copy(ioutil.Discard, res.Body)
```
