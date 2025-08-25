### 判断interface是否为nil

---

当类型和值的部分都是nil，interface才为空。
这里判断值部分是否为空
```go
package main

import (
	"fmt"
	"reflect"
)

func IsNil(x interface{}) bool {
  if x == nil {
    return true
  }

  return reflect.ValueOf(x).IsNil()
}

func main() {
	var x interface{}
	var y *int = nil
	x = y
	fmt.Println(IsNil(x))
}
```