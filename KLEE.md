# KLEE 论文阅读

## Overview  

通过MINIX’s tr tool举例，说明两个问题，即Complexity和Environment Dependency。对于Complexity，tr中的代码比较复杂，而且不容易读懂，边界条件不明郎，控制流不容易跟踪等。Environment Dependency表示程序不仅依赖于函数参数，还依赖于外部文件接口的函数调用，测试到所有重要的值以及边界问题很困难。

### KLEE如何使用
1. 首先通过llvm前端，将C代码转换成bytecode，命令：llvm-gcc --emit-llvm -c tr.c -o tr.bc
2. 使用Klee命令进行符号执行 命令：klee --max-time 2 --sym-args 1 10 10 --sym-files 2 2000 --max-fail 1 tr.bc   
参数说明
    - --max-time 2：tr.bc的整个符号执行的时间是2分钟
    - --sym-args 1 10 10：包含三个符号化参数，长度分别为1个字符，10个字符，10个字符。
    - --sym-files 2 2000：分别使用标准的输入和一个文件，并且每个文件的大小为2000 bytes的符号化数据。

### 接下来给出了KLEE对于tr工具进行符号执行的具体过程  
具体的就不说了  

## KLEE架构
KLEE是对原来的EXE进行的重新设计，在上层，KLEE充当符号执行程序的操作系统进程和解释器。每个符号执行进程都包含寄存器文件，栈，堆，程序计数器和路径约束。这这里为了与传统的进程进行区分，在这里将符号进程叫做state。

### Basic Architecture
在同一时刻，KLEE可能会执行很多state（进程），KLEE的核心是一个解释器，他会循环的执行所有的state，直到所有的state都结束。  
与不同进程不同，state通过表达式的方式存储寄存器、栈、堆数据而不是数据值，同时表达式是由符号变量组成的。  
条件分支采用布尔表达式（分支条件）并根据条件是真还是假来更改state的指令指针。对于不同的分支，KLEE通过复制state的方式来同时探索两个路径，并在每个路径约束上更新指令指针。
对于可能出现错误的操作，例如除零操作，KLEE也会生成一个分支来检查是否除零，这个分支与普通的分支采用同样的处理方式。如果发现了错误，KLEE生成测试用例，触发错误并结束这个state。  
对于load和store指令，KLEE会判断是否出现越界，KLEE将load和store指令映射为对数组的读写表达式。   
对于指针操作是非常困难的，因为一个指针可能被refer成很多不同的对象。为了简单期间，KLEE采用了如下的做法，如果一个指针可以被解引用成N个不同的对象，KLEE复制当前的state N次，这种方式虽然非常浪费资源，但是在符号执行中并不常见。

### Compact State Representation（压缩表示state空间）
采用copy-on-write的方式减少每个state的内存需求。通过将heap实现为一个不可变的map，可以实现不同state之间对heap的共享。

### Query Optimization（查询优化）
在将query输入STP进行求解之前，一定要其进行优化，主要的优化方法：
- Expression Rewriting
- Constraint Set Simplification
- Implied Value Concretization
- Constraint Independence
- Counter-example Cache  

KLEE使用这些优化手段，同时还做了实验来分析这些优化手段对性能产生的影响。

### State scheduling（进程调度）
- Random Path Selection 
- Coverage-Optimized Search 

## Environment Modeling（环境建模）
当代码从命令行、环境变量、文件数据、网络数据时，理论上，我们希望这些值都能合法（symbolic data）的产生，而不是一个具体的数据。当对这些环境进行写操作的时候，也希望这些写操作能够对后面的读操作有影响，这些特点就需要检查程序能够探索出所有的活动并且没有错误出现。  
KLEE采用了将堆环境的访问转化到访问自己写的环境模块中。这些自己写的模块能够很好的产生出想要的约束。KLEE实现了关于文件接口的相关环境模型接口。

### Example: modeling the file system
对于具体文件，直接调用POSIX接口执行系统调用，对于符号文件，则模拟符号文件系统的对操作产生的影响。我已经看了这些代码，有一定的了解。但是有个问题，符号文件是如何构建的？
### Failing system calls
实际的环境中，可能出现意想不到的失败，例如向一个已经满的磁盘空间写数据。这些错误可能导致意想不到的问题并且很难发现。为了能够发现这些bug，KLEE通过使系统调用失败来模拟这些环境产生的问题。
### Rerunning test cases
KLEE提供了重新运行测试用例的工具。每个测试用例描述了测试环境中的符号执行的一个实例。

## Evaluation


