# Meltdown 漏洞实验报告

## 环境搭建

实验的第一步是搭建环境，即可以被 Spectre 漏洞攻击的系统。首先，我选择了 Ubuntu 作为实验系统。Ubuntu 的 [Wiki](https://wiki.ubuntu.com/SecurityTeam/KnowledgeBase/SpectreAndMeltdown) 写明了不同发行版受漏洞影响的情况，其中 Ubuntu 14.04 在不安装更新的情况下是会受到 Meltdown 和 Spectre 攻击的。因此，我选择 Ubuntu 14.04 amd64 作为实验系统。

最终的实验环境为：
* Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz
* Ubuntu 14.04 amd64
* gcc 4.8

## Meltdown 原理

* 乱序执行

    现代 CPU 在执行指令时大都采取乱序执行的方式以提升执行效率。
    当控制流发生变化时，某些在原始程序语义下不应该被执行的指令此时已经执行并产生了一些副作用。
    CPU 会撤销那些指令在内存、寄存器等层面造成的影响，但不会撤销它们对 cache，ALU 等 CPU 内部模块产生的副作用。


* Flush/Reload 旁信道攻击
    
    我们可以通过旁信道攻击的方式，捕获上述副作用。
    例如，我们想越权访问某个地址，那么我们可以先把它取出来，记为变量 x，然后我们开辟256个内存页（记为pages），并访问第 x 页。
    这段代码的副作用是，pages 的第 x 页被放入了缓存。这时，我们可以逐个访问这些页，并记录访问时间。
    我们可以通过判定某个页是否在缓存中来推算出 x，以实现越权访问。
    其中，flush 指令可用于从 cache 移除某地址的内容，因而尽可能地保证了我们判断的正确性。


* CPU 记时

    CPU 提供了 rdtsc 指令，可以用来进行高精度记时。


## 程序结构

1. 对汇编指令的封装（来自 libkdump，做了简单的修改以使它更清晰）
    * maccess
    * flush
    * flush_reload
    * flush_reload_time
1. libkdump 中重要的工具函数
    * detect_flush_reload_threshold
    * segfault_handler
1. meltdown 主程序
    * meltdown_try_read 通过 meltdown 攻击，读取一个内存地址的值
    * meltdown_read 对 meltdown_try_read 读取的结果进行统计，选出最有可能的值
1. 测试代码

第一部分是对汇编代码的封装，较为简单易懂。

第二部分是 libkdump 库中的两个工具函数，detect_flush_reload_threshold 通过 flush/reload 的方法，
计算出 cache hit 的期望时间，即，若某次访问的时间小于该时间，则视为 cache 命中。
segfault_handler 是异常处理函数，用于捕获段错误。在捕获到段错误之后，segfault_handler 会忽略它，并恢复正常控制流。

第三部分是 meltdown 的主程序，这里一共有两个版本，一个是 libkdump 实现的版本，一个是我自己实现的版本。
libkdump 通过 detect_flush_reload_threshold 计算 cache 命中的期望时间，然后通过多次循环（10000 数量级），找到 cache 命中的页。
之后经过很少的几次统计，输出最终结果。另一个是我实现的版本，首先通过 flush_reload_time 获取每页访问时间，并认为访问时间最短的页为 cache 命中的页，然后进行统计，找到命中率最高的页，若命中率高于阈值，则输出结果，若低于阈值，则再进行一轮统计（一轮大概1000次访问）。

第四部分是测试代码，运行结果为：
```
official method:        Hey Jude, don't be afraid. You were made to go out and get her.
my method:              Hey Jude, don't be afraid. You were made to go out and get her.
```

## 测试方法
```
make
./lab4
```

## 讨论与备注

* libkdump 中有针对各种处理器架构的不同处理策略，我只选择了其中一种组合(signal handler + x86_64 + rdtsc) 进行实验。
* 官方实现的方法容易陷入长时间阻塞，我实现的方法准确率稍逊于官方实现。
* 对物理内存的访问可在内核地址空间内进行。通过读取页表信息，可实现攻击任意进程。
* 攻击者开辟的内存 pages 需要填充相同的内容，这可以大幅提高攻击准确度。
* libkdump 中创建了一些 load thread，并在某些时刻将控制权转交给这些线程。这不是必要的，但可以提高攻击准确度。

## 参考资料

1. [Meltdown 论文](https://meltdownattack.com/meltdown.pdf)
1. [Meltdown wiki](https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability))
1. [Ubuntu Wiki](https://wiki.ubuntu.com/SecurityTeam/KnowledgeBase/SpectreAndMeltdown)
1. [Meltdown 官方demo](https://github.com/IAIK/meltdown)
1. [Meltdown 漏洞检测脚本](https://github.com/speed47/spectre-meltdown-checker)