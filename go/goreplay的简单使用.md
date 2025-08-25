### goreplay的简单使用

---

```shell
# 捕获服务器流量到本地
./gor --input-raw :8082 --output-file=requests.gor
 
# 从本地文件回放流量
./gor --input-file requests_0.gor --output-http="http://172.16.106.237:8082"
 
# 实时将流量请求打印到终端
./gor --input-raw :8082 --output-stdout
 
# 复制流量实时转发到另一台服务器
./gor --input-raw :8082 --output-http="http://172.16.106.237:8082"
 
# 放大流量
./gor --input-file "requests.gor|200%" --output-http="http://172.16.106.237:8082"
 
# 缩小流量
./gor --input-file "requests.gor|20%" --output-http="http://172.16.106.237:8082"
```