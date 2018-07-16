# Datasize-Aware High Dimensional Configurations Auto-Tuning of In-Memory Cluster Computing
## 摘要
### 专业术语
IMC：In-Memory cluster Computing, 代表是Spark<br>
ODC: On-Disk cluster Computimng, 代表是MapReduce/Hadoop，Dryad<br>
Cluster Computering: 集群计算<br>
periodic long job：daily，weekly computing，如以周为周期的计算。
首先介绍了IMC比ODC所拥有的性能优势，10X；但是IMC同样存在着挑战，主要包含两个方面：<br>
- IMC的性能与输入数据数组的大小有很大关系，但是却很难将其加入到性能模型中去；
- IMC中关键性能的配置参数很多，40多个，这也需要很复杂的数学模型才能达到高精度。
    
解决方法简介：<br>
    作者提出了DAC方法，针对于给定的IMC程序，能够确定高维度配置来优化集群计算。它支持将数据数组作为参数，同时性能参数的限制可以达到41个之多。采用的方法包含两个方面：
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
