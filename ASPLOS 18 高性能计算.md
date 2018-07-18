# Datasize-Aware High Dimensional Configurations Auto-Tuning of In-Memory Cluster Computing
## 摘要
### 专业术语
IMC：In-Memory cluster Computing, 代表是Spark<br>
ODC: On-Disk cluster Computimng, 代表是MapReduce/Hadoop，Dryad<br>
Cluster Computering: 集群计算<br>
periodic long job：daily，weekly computing，如以周为周期的计算。<br>
RDD：Resilient Distributed Datasets<br>
DAG: Directed acyclic graph<br>
DAC: Datasize-aware auto-tuning Computing.<br>
首先介绍了IMC比ODC所拥有的性能优势，10X；但是IMC同样存在着挑战，主要包含两个方面：<br>
- IMC的性能与输入数据数组的大小有很大关系，但是却很难将其加入到性能模型中去；*是将整个数组作为参数，还是只是数组的大小呢？*
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
- 定义：DAC是一种配置调整方法，针对于一个给定集群上的指定的Spark程序，它可以自动的调节配置参数来优化性能。
- Spark program：使用相近的数据集作为输入，重复的运行多次程序。但是这些数据集的内容是不同的。
![DAC 工作流程](/images/DAC_Block_Diagram.png)<br>
上图中，介绍了DAC的三个组成部件，其中Collecting生成一些配置信息，通过生成的配置自动的运行IMC程序，并收集执行时间相关经验; modeling部件负责构建performance模型，它将高维配置参数和输入的数据集size作为参数，形成一个function。主要创新是DAC可以生成大量参数的performance模型，这是以前的自动调节方法没有的; searching部件自动的查找能够生成最秀performance的配置。整体上，modeling部件依赖于collecting部件的结果，searching部件通过modeling部件的输出结果中选择最佳配置。
- Collecting Data
    收集数据的目的是为了根据收集信息在Modeling部件中构建精准性能模型。对于给定的Spark程序，主要收集的性能数据包括如下：<br>
    - configuration信息：作者开发了一个CG(Configuration Generator)，负责生成配置信息，它以一个向量的方式来表示。conf*i* = {c*i1*,c*i2*,...,c*ij*,...,c*in*}。其中n=41，代表有41个配置参数，这些参数可以从table 2中获取。
    - Dataset信息：通过DG(dataset generator)生成m个输入dataset，这m个dataset包含不等的大小，size大小的范围最少要大于10%。m的默认值设置为10
    - program-pairs: 将10个dataset和对应的程序组成程序对。
    使用K个不同的configuration信息，运行10个程序对；当一个程序对结束后，创建一个向量存储执行时间和相应的配置。
- Modeling Performance
    对performance进行建模，针对于响应面，人工神经网络，支持向量机和随机深林算法在构建精准模型的时候都失败了。作者认为高维度配置参数和数据集输入带来了建模的复杂性。over-fitting问题在统计推理和机器学习算法中都是普遍存在的，所以作者提出了Hierarchical Modeling方法。其关键思想是通过多个简单模型来预测performance，而不是通过一个复杂的模型来预测。下面是对该方法的解释，不想看了。以后有机会再看。但是给出了思想，就是通过多个简单模型来代替单个复杂模型。
- Searching Optimal Configuration
    有很多搜索复杂配置空间的算法，如递归随机查找，模式匹配和遗传算法等。最终，作者选择了遗传算法作为复杂配置空间搜索算法, 相关描述如下图：
![搜索最优配置](/images/Config_Searching.png)

- 实现
    - 使用了R语言实现CG
    - CG生成随机值后，将随机值和配置参数写入到配置文件spark-dac.conf中。
    - 接下，根据配置文件执行程序，当执行完成后，收集执行时间，并将其与参数配置，input dataset组成定义中定义的向量存储。重复这一过程来收集训练数据集。通过算法1来构建性能模型，也是用R来实现的。最后，通过GA来完成最优的配置的选择。

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

## 总结

针对于cluster计算的性能问题，作者提出了一种设计方案，来提高性能。这种方案的主旨是构建精确的performance模型和选择最优配置方案。其中，通过构建简单多级的performance模型取代构建复杂的单一计算模型，并使用了遗传算法来选择最优配置方案。通过实验和分析，证明了该方案在性能上有显著提升。本文包含大量的实验和分析工作。看来好文章必须做好实验和分析环节。
