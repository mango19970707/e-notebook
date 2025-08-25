### 优化GC

---

#### 问题描述

程序频繁触发GC，严重影响程序的性能。

- 通过设置环境变量“GODEBUG”开启GC日志
```shell
GODEBUG=gctrace=1
```

- 日志如下：
```shell
gc 3 @46.886s 6%: 0.006+39+0.004 ms clock, 0.006+36/2.2/0.8+0.004 ms cpu, 420->435->210 MB, 420 MB goal, 1 P
```

- 参数含义

|     Key     |                         Description                          |
|:-----------:|:------------------------------------------------------------:|
|    `gc 3`    | 发生GC的次数 |
|  `@46.886s`   |             程序执行的总时间             |
|    `6%`     | gc时时间和程序运行总时间的百分比 |
| `0.006+39+0.004 ms clock` |表示第一次STW + Mark assist/Mark(Dedicated + Fractional)/Mark(Idle) + 第二次STW的CPU时间。与时钟时间的统计不同，CPU时间会对各个核上对应的处理时间进行累加。比如0.006+36/2.2/0+0.004 ms cpu，0.006ms表示第一次STW过程中，被STW的多个核的时钟时间之和，其值大于等于对应时钟时间。36ms表示在整个Mark过程中，进行assist Mark的CPU累计时间。2.2ms表示在整个Mark过程中，在gcMarkWorkerDedicatedMode和gcMarkWorkerFractionalMode两种工作模式下进行Mark处理的CPU累计时间之和。0.8ms表示在gcMarkWorkerIdleMode模式下进行Mark处理的CPU累计时间之和。0.004ms表示第二次被STW的多个核的时钟时间之和； |
| `0.006+36/2.2/0.8+0.004 ms cpu` | 表示第一次STW + Mark assist/Mark(Dedicated + Fractional)/Mark(Idle) + 第二次STW的CPU时间。与时钟时间的统计不同，CPU时间会对各个核上对应的处理时间进行累加。比如0.006+36/2.2/0+0.004 ms cpu，0.006ms表示第一次STW过程中，被STW的多个核的时钟时间之和，其值大于等于对应时钟时间。36ms表示在整个Mark过程中，进行assist Mark的CPU累计时间。2.2ms表示在整个Mark过程中，在gcMarkWorkerDedicatedMode和gcMarkWorkerFractionalMode两种工作模式下进行Mark处理的CPU累计时间之和。0.8ms表示在gcMarkWorkerIdleMode模式下进行Mark处理的CPU累计时间之和。0.004ms表示第二次被STW的多个核的时钟时间之和； |
| `420->435->210 MB` | 表示GC开始前申请的内存大小 -> GC标记(Mark)结束后申请的内存大小 -> 被标记存活的内存大小。比如420->435->210 MB，表示GC开始前一共申请了420MB的内存，GC标记(Mark)处理完后一共申请了435MB的内存，也就说在整个标记阶段，又新申请了15MB的内存，标记阶段一共标记了210MB的内存，就是说有435MB-210MB=225MB的内存可以被回收； |
| `420 MB goal`  |               目标堆大小               |
|   `1 P`    |                  占用核个数                  |


- 总结，从上述gc日志来看，程序到了420MB的时候就触发了GC，那么现在的问题就是如何把GC的触发点变大。

### 使用GOGC把GC的触发点变大
GOGC默认值是100，例如：你程序的上一次GC完，驻留内存是100MB，由于你GOGC设置的是100，所以下次你的内存达到200MB的时候就会触发一次GC，如果你GOGC设置的是200，那么下次你的内存达到300MB的时候就会触发GC。

```shell
export GOGC=值
go run your_program.go
```