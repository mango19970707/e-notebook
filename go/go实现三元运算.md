### 实现三元运算

---

```go
package main

import "fmt"

func Ter[T any](cond bool, a, b T) T {
	if cond {
		return a
	}

	return b
}

func main() {
	fmt.Println(Ter(true, 1, 2))  // 1
	fmt.Println(Ter(false, 1, 2)) // 2
}
```