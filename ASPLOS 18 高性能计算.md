# Datasize-Aware High Dimensional Configurations Auto-Tuning of In-Memory Cluster Computing
## 摘要
### 专业术语
IMC：In-Memory cluster Computing, 代表是Spark<br>
ODC: On-Disk cluster Computimng, 代表是MapReduce/Hadoop，Dryad<br>
Cluster Computering: 集群计算<br>
periodic long job：daily，weekly computing，如以周为周期的计算。<br>
RDD：Resilient Distributed Datasets
DAG: Directed acyclic graph
首先介绍了IMC比ODC所拥有的性能优势，10X；但是IMC同样存在着挑战，主要包含两个方面：<br>
- IMC的性能与输入数据数组的大小有很大关系，但是却很难将其加入到性能模型中去；
- IMC中关键性能的配置参数很多，40多个，这也需要很复杂的数学模型才能达到高精度。
    
解决方法简介：<br>
    作者提出了DAC方法，针对于给定的IMC程序，能够确定高维度配置来优化集群计算。它支持将数据数组作为参数，同时性能参数的上限可以达到41个之多。采用的方法包含两个方面：
- 采用了分级建模，结合多个独立子模型的方式；
    针对于建立多个简单的精确模型，比建立一个大的复杂的模型要容易的多，HM构建了一阶、二阶或者多阶分级模型
- 采用遗传算法来查找优化配置方式。

---
## 介绍
- IMC参数多造成的问题，解决方法
    - 参数的组合会产生大量的组合元组
    - 累积的运行时间太长--每个IMC程序的运行时间可能需要花费积分到一个小时的时间。
    - 解决方法：常用的解决方法是通过建立performance模型，并给定一个configuration，来预测给定参数的情况下的运行时间，这种方法被大量用在ODC和传统的分布式系统中。该方法的关键是要有精确的performance模型，否则优化的效果不佳。
但是，往往由于摘要中提到的两点因素，对于构建精确的performance模型非常困难。

## Background
- Spark
    Spark构架与MapReduce相似，但是不能直接在Hadoop类似的构架中使用。主要原因为Spark采用了一种数据结构RDDs在内存中保存中间可复用运行结果。RDDs实际上是一种集群内存的抽象，允许用户显示的控制分区来优化数据存放位置，也可允许通过一组如map，hash-jion等操作来管理它。这里的partion是指对运行中间结果的分区还是对job代码的分区呢？<br>
![image](https://github.com/mancanfly-1/work_records/blob/master/images/spark_workflows.png)
    1. 首先，基于Spark应用程序生成DAG(directed acyclic graphic)；
    2. 其次DAG被分割成几个阶段，每个阶段中包含一组并行任务。 每个任务对应一个RDD partition，分别计算各自的任务的计算结果。一个Spark job可以分为多个stage，每个stage可能依赖于其他的stage。这个依赖性叫做lineage，存储在RDD中
    3. 通过executor执行每个stage中的任务，每个executor能够使用的系统资源是通过配置参数限定的。例如：可以通过设置spark.executor.meomry来针对一个executor进行内存设置；这段内存还可以通过spark.memory.fraction来详细划分出执行内存、用户内存和保留内存等。
    4. 一个spark任务要通过160多个配置参数来进行控制。他们被分为14个部分，包括：application, runtime environment, shuffle behavior, data serialization, memory management, execution behavior, networking, Spark UI, scheduling, dynamic allocation, secu- rity, encryption, sparkstreaming, and sparkR。
    5. 在这些参数中，有41个参数可以被很容易的调整，并且严重的影响性能。所以，文章中针对于这41个参数进行tuning
- 动机
    回答了两个问题：1. Spark之类的IMC是否采用了足够多的参数；2. 当前ODC的性能建模技术能否应用到IMC中？
    - 输入数据集大小对性能造成的敏感性
        通过实验显示，程序的执行时间对数据集大小非常敏感。主要原因是IMC和ODC的本质差别为，由于IMC通过在存储器中防止尽可能多的数据，IMC的性能更高，但是，轻微的扰动就会对计算产生巨大影响。<br>
    - ODC 建模技术的缺陷
        都有哪些技术可以在ODC建模中使用？
        分析建模、统计推理和机器学习技术在构建性能模型中被使用，他们主要用于配置参数。通过实验分析得到，已经存在的建模技术，对于数据集和41个配置参数作为输入参数时，当前这些技术不能够精确的构建性能模型。

## DAC 方法

## 实验方法
    实验平台包含六个DELL服务器，一个master node，五个slave node。每台服务器是12个处理器，每个处理器6个核心，64G内存。OS是libnux enterprise server 12. spark为V1.6

### 测试程序的选择

Application | Abbr | input datasize
---|---|---
PageRank    | PR | 1.2, 1.4, 1.6, 1.8, 2 (million pages)
KMeans      | KM | 160, 192, 224, 256, 288 (million points)
Bayes       | BA | 1.2, 1.4, 1.6, 1.8, 2 (million pages)
NWeight     | NW | 10.5, 11.5, 12.5, 13.5, 14.5 (million edges)
WordCount   | WC | 80, 100, 120, 140, 160 (GB)
TeraSort    | TS | 10, 20, 30, 40, 50 (GB)
### 配置参数设置
