## 1：来源：
[Dynamic Data Race Detection in Go Code](https://www.uber.com/en-CA/blog/dynamic-data-race-detection-in-go-code/)
[Data Race Patterns in Go](https://eng.uber.com/data-race-patterns-in-go/)
## 2：Go语言内置 data race detector
并发程序不好开发，更难于调试。并发是问题的滋生地，即便Go内置并发并提供了基于CSP并发模型的并发原语(goroutine、channel和select)，实际证明，[现实世界中，Go程序带来的并发问题并没有因此减少](https://songlh.github.io/paper/go-study.pdf)。**“没有银弹”再一次应验**！
不过Go核心团队早已意识到了这一点，在[Go 1.1版本](https://go.dev/doc/go1.1#race)中就为Go工具增加了race detector，通过在执行go工具命令时加入-race，该detector可以发现程序中因对同一变量的并发访问(至少一个访问是写操作)而引发潜在并发错误的地方。Go标准库也是引入race detector后的受益者。race detector曾[帮助Go标准库检测出42个数据竞争问题](https://go.dev/blog/race-detector)。
race detector基于Google一个团队开发的工具[Thread Sanitizer(TSan)](https://github.com/google/sanitizers)(除了thread sanitizer，google还有一堆sanitizer，比如：AddressSanitizer, LeakSanitizer, MemorySanitizer等)。第一版TSan的实现发布于2009年，其使用的检测算法“源于”老牌工具Valgrind。出世后，TSan就帮助Chromium浏览器团队找出近200个潜在的并发问题，不过第一版TSan有一个最大的问题，那就是**慢！**。
因为有了成绩，开发团队决定重写TSan，于是就有了v2版本。与V1版本相比，v2版本有几个主要变化：

- 编译期注入代码(instrumentation)；
- 重新实现运行时库，并内置到编译器(LLVM和GCC)中；
- 除了可以做数据竞争(data race)检测外，还可以检测死锁、加锁状态下的锁释放等问题；
- 与V1版本相比，v2版本性能提升约20倍；
- 支持Go语言。
## 3：ThreadSanitizer v2版本工作原理
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663461498604-f37c111c-9885-438e-b086-bd34c02d4321.png#clientId=u966bb013-0c92-4&errorMessage=unknown%20error&from=paste&height=98&id=u1da45001&name=image.png&originHeight=196&originWidth=1024&originalType=binary&ratio=1&rotation=0&showTitle=false&size=45502&status=error&style=none&taskId=u510e922e-c440-4942-99ec-58bcc3e3c91&title=&width=512)
