# Alluxio-Internal
经过近期对alluxio的研究，本着自身学习记录，同时因为当前对alluxio似乎还没有比较全面、深入的分析。这边系统的对alluxio进行源码分析，同时给出在工作过程中遇到的坑，和解决的坑。alluxio使用的源码是1.4.0。

##简单介绍

alluxio和hdfs有些类似、都是分布式的文件系统，hdfs基于磁盘介质存储、alluxio基于内存介质存储；hdfs基于replica方式进行容错、alluixo基于lineage的方式进行容错（目前容错性处于test阶段，并不完善，建议重要数据还是需要持久化到底层的文件系统）；alluxio和hdfs都有类似的文件操作api、类似的shell命令（目前alluixo并没分admin和非admin命令）；alluxio和hdfs都是基于文件块的形式存储数据，都是典型的master slave集群架构，都有master ha等。同时它们也有很多细节的区别：alluixo RPC使用thrift，hdfs使用的是protobuf；alluxio更多的用来加速上层计算框架，hdfs则更多的用来持久化存储；alluxio对上层的计算框架的locality不如hdfs等。

##主要内容
对alluixo进行比较全面的分析，将从以下几个方面着手。

1. [Build And Deploy](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) 编译部署alluxio
2. [How to use alluxio](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio的使用 (未写)
3. [RPC Thrift](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio RPC底层thrift介绍 (未写)
4. [Alluxio RPC](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio RPC源码分析 (未写)
5. [Alluxio Client](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio Client源码分析 (未写)
6. [Alluxio Master](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio Master源码分析 (未写)
7. [Alluxio Worker](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio Worker源码分析 (未写)
8. [Alluxio security](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) alluxio 认证授权源码分析 (未写)
9. [Alluxio bug and fix bug](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) 挖坑、填坑 (未写)
10. [Configure](https://github.com/gjhkael/Alluxio-Internal/blob/master/Build-And-Deploy.md) 配置与源码分析 (未写)
