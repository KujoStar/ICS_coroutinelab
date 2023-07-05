# 计算机系统概论 2022 秋 协程实验

在本实验中，同学们将亲手借助汇编/内嵌汇编的方式，在熟悉 x86_64 系统架构的前提下，完成一个简单的用户态的有栈协程库，并利用协程库进行睡眠排序（sleep_sort）的编写以及二分查找的优化。

本次实验共分为三个子任务，同学需要依次完成三个子任务并编写实验报告。

三个子任务如下：

1. 完成协程库的编写，并通过基础测试。
2. 向协程库添加 sleep 函数，并能够通过 sleep_sort 测试。
3. 使用协程库对二分查找进行优化，并报告优化结果。

你需要在 REPORT.md 文件中以 Markdown 格式编写实验报告，然后把整个目录压缩成 zip 格式再提交到网络学堂。

## 环境配置

本实验的运行环境是 x86_64 Linux。你可以在课程提供的远程 Linux 服务器上完成本实验。

编译的时候需要 GCC 和 Make。在根目录下运行 `make` 即可编译，生成的二进制在 `bin` 目录下。

## Task 1: 协程库的编写

你需要认真阅读框架代码并以及代码中的注释并完成标有 `TODO: Task 1` 字样的函数的编写。

我们建议分为两个小步骤来实现这一部分：

1. 实现 `coroutine_pool::serial_execute_all` 函数中启动协程的部分以及 `context.S` 中切换协程的汇编代码。完成这两部分之后可以自己实现一个不带 yield 的“协程库”。在此基础上可以进行一个简单的测试以检查协程栈是否正常分配在堆上，协程返回的时候是否正常等。
2. 实现 `yield` 以及完善 `coroutine_pool::serial_execute_all`。完成协程的 yield 以及 resume 和重新调用。

关于这一部分的实现细节在代码注释中有比较详细的内容，如果你认为实现有困难可以参考代码中给出的注释提示。

实验报告的额外要求：

1. 绘制出在协程切换时，栈的变化过程；
2. 并结合源代码，解释协程是如何开始执行的，包括 `coroutine_entry` 和 `coroutine_main` 函数以及初始的协程状态；

完成 Task 1 以后，应该可以正常运行 `bin/sample` 程序，并得到下面的输出：

```
in show(): 0
in show(): 0
in show(): 1
in show(): 1
in show(): 2
in show(): 2
in show(): 3
in show(): 3
in show(): 4
in show(): 4
in main(): 0
in main(): 0
in main(): 1
in main(): 1
in main(): 2
in main(): 2
in main(): 3
in main(): 3
in main(): 4
in main(): 4
```

## Task 2: 实现 sleep 函数

在协程中，不能使用操作系统提供的 sleep 函数，因为它会阻塞整个线程，但希望的效果是切换到其他协程，等到 sleep 时间结束后，再继续执行协程。

因此，协程库也会提供一个 sleep 函数，它的实现方法是，标记当前协程 `ready = false`，并注册一个 `ready_func`，它会检查当前的时间，是否已经超过了应该继续执行的时间，然后进行 `yield`。那么，`coroutine_pool::serial_execute_all` 就需要判断协程的当前状态，如果它 `ready == true`，说明可以继续执行；如果它 `ready == false`，则调用 `ready_func`，如果返回 `true`，说明可以继续执行了，就设置 `ready = true` 并切换到协程。

你需要实现 `sleep` 函数，具体定义可以参照代码注释，并修改 `coroutine_pool::serial_execute_all`，实现对 `ready` 的判断。并完成 sleep_sort 的测试。

**提示**: 你可以使用 `parallel_execute_all` 与 `serial_execute_all` 进行对比。

实验报告的额外要求：

1. 按照时间线，绘制出 `sleep_sort` 中不同协程的运行情况；
2. 目前的协程库实现方式是轮询 `ready_func` 是否等于 `true`，设计一下，能否有更加高效的方法。

完成 Task 2 以后，应该可以正常运行 `bin/sleep_sort` 程序，可以输入一组数，程序会从小到大输出排序后的数。下面是输入 1, 3, 4, 5, 2 的例子：

```
$ ./bin/sleep_sort
5
1 3 4 5 2
1
2
3
4
5
```

## Task 3: 利用协程优化二分查找

在数据量比较大的时候，二分查找会产生大量的缓存缺失，而从内存读取数据到 CPU 需要花费几百个 CPU 周期。因此，可以利用协程来优化二分查找。优化方法是，修改二分查找中容易产生缓存缺失的代码，改为先使用预取指令，让 CPU 异步地读取数据，紧接着调用 `yield` 来切换到其他协程。当多次切换后返回到刚才使用预取指令的协程的时候，CPU 已经把数据读取到了缓存中，此时就节省了很多时间。

你需要修改 `lookup_coroutine` 函数，插入代码，来实现上面所述的优化算法，并在实验报告中汇报性能的提升效果。你可以通过命令行参数设置不同的参数，观察不同参数下的性能。

实验报告的额外要求：

1. 汇报性能的提升效果。

可以参考下列文献：

Georgios Psaropoulos, Thomas Legler, Norman May, and Anastasia Ailamaki. 2017. Interleaving with coroutines: a practical approach for robust index joins. Proc. VLDB Endow. 11, 2 (October 2017), 230–242. https://doi.org/10.14778/3149193.3149202

## 实验报告

你需要在 REPORT.md 文件中以 Markdown 格式编写实验报告，内容包括：

1. 姓名，学号和班级；
2. 三个小节，分别记录你为了实现三个子任务所添加到代码，并对代码进行详细的注释；每个子任务都有实验报告的额外要求，请阅读上面的文本；
3. 记录你在完成本实验的过程中，与哪些同学进行了交流，查阅了哪些网站或者代码；
4. 总结和感想。