### channel应用场景总结

--- 

- ***信号传递***
有 4 个 goroutine，编号为 1、2、3、4。每秒钟会有一个 goroutine 打印出它自己的编号，要求你编写程序，让输出的编号总是按照 1、2、3、4、1、2、3、4……这个顺序打印出来。
```go
type Token struct{}
 
func newWorker(id int, ch chan Token, nextCh chan Token) {
    for {
        token := <-ch         // 取得令牌
        fmt.Println((id + 1)) // id从1开始
        time.Sleep(time.Second)
        nextCh <- token
    }
}
func main() {
    chs := []chan Token{make(chan Token), make(chan Token), make(chan Token), make(chan Token)}
 
    // 创建4个worker
    for i := 0; i < 4; i++ {
        go newWorker(i, chs[i], chs[(i+1)%4])
    }
 
    //首先把令牌交给第一个worker
    chs[0] <- struct{}{}
   
    select {}
}
```

- ***信号通知***
  使用 chan 实现程序的 graceful shutdown，在退出之前执行一些连接关闭、文件 close、缓存落盘等一些动作。
```go
func main() {
    var closing = make(chan struct{})
    var closed = make(chan struct{})
 
    go func() {
        // 模拟业务处理
        for {
            select {
            case <-closing:
                return
            default:
                // ....... 业务计算
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()
 
    // 处理CTRL+C等中断信号
    termChan := make(chan os.Signal)
    signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
    <-termChan
 
    close(closing)
    // 执行退出之前的清理动作
    go doCleanup(closed)
 
    select {
    case <-closed:
    case <-time.After(time.Second):
        fmt.Println("清理超时，不等了")
    }
    fmt.Println("优雅退出")
}
 
func doCleanup(closed chan struct{}) {
    time.Sleep((time.Minute))
    close(closed)
}
```

- ***使用channel的声明控制读写权限***
```go
// 只有generator进行对outCh进行写操作，返回声明
// <-chan int，可以防止其他协程乱用此通道，造成隐藏bug
func generator(int n) <-chan int {
    outCh := make(chan int)
    go func(){
        for i:=0;i<n;i++{
            outCh<-i
        }
    }()
    return outCh
}
  
// consumer只读inCh的数据，声明为<-chan int
// 可以防止它向inCh写数据
func consumer(inCh <-chan int) {
    for x := range inCh {
        fmt.Println(x)
    }
}
```

- ***超时控制***
```go
func doWithTimeOut(timeout time.Duration) (int, error) {
    select {
    case ret := <-do():
        return ret, nil
    case <-time.After(timeout):
        return 0, errors.New("timeout")
    }
}
  
func do() <-chan int {
    outCh := make(chan int)
    go func() {
        // do work
    }()
    return outCh
}
```

- ***定时任务***
```go
func main() {
    timer := time.NewTicker(time.Second)
    for {
        select {
        case <-timer.C:
            fmt.Println("执行了")  // 每隔1秒执行一次
        }
    }
}
```

- ***控制并发数量***
```go
var limit = make(chan int, 3)
 
func main() {
    // 通过channel控制最大并发数量
    tasks := [...]int{11, 22, 33, 44, 55, 66, 77, 88, 99, 100}
    for i, v := range tasks {
        // 为每一个任务开启一个goroutine
        go func(i, v int) {
            // 通过channel控制goroutine最大并发数量
            limit <- -1
            fmt.Println(i, v)
            time.Sleep(time.Second)
            <-limit
        }(i, v)
    }
    time.Sleep(time.Second * 4)
}
```

- ***生产者消费者模型***
```go
func main() {
    tasksChan := make(chan int, 100)  // 任务队列
    go workerTask(tasksChan)  // 开启处理任务的协程池
    // 发起任务
    for i := 0; i < 10; i++ {
        tasksChan <- i
    }
    select {
    case <-time.After(time.Second * 3):
    }
}
func workerTask(tasksChan chan int) {
    // 开启5个协程去处理任务队列中的数据
    GOS := 5
    for i := 0; i < GOS; i++ {
        // 局部变量在堆栈上存储，也是变量逃逸的一种场景(解决方法：使用闭包)
        go func(i int) {
            for {
                value := <-tasksChan
                fmt.Printf("finish task: %d by worker %d\n", value, i)
                time.Sleep(time.Second)
            }
        }(i)
    }
}
```