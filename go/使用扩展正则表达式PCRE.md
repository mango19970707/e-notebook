### 使用扩展正则表达式PCRE

---

- 安装libpcre++-dev
```shell
# mac环境
brew --prefix pcre
export CGO_CFLAGS="-I$(brew --prefix pcre)/include"
export CGO_LDFLAGS="-L$(brew --prefix pcre)/lib -lpcre"

# debian环境
sudo apt-get install libpcre++-dev
```

- 使用示例
```go
package main
 
import (
    "fmt"
    "github.com/glenn-brown/golang-pkg-pcre/src/pkg/pcre"
)
 
func main() {
    pattern := `^(50.16|50.18)`
    target := []byte(`50.19.23.23`)
    re := pcre.MustCompile(pattern, 0)
    res := re.Matcher(target, 0)
    fmt.Println(res.Matches())
}
```