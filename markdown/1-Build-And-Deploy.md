#Build-and-install-Alluxio

##编译alluxio

编译alluxio很简单如下：
```
git clone https://github.com/Alluxio/alluxio.git
git checkout -t  origins/branch-1.4
mvn clean package -Pspark -Dhadoop.version=2.6.0 -DskipTests
```
补充说明：

* 这种开源多分支项目，可以先使用git branch -a看一下所有的分支、然后使用checkout -t来切换到某个远程分支，有兴趣的同学可以自己百度如何切换到远程分支。
* 如果对alluxio源码进行了修改，编译之后将assembly下的alluxio jar替换线上集群相应的jar即可，不需要完全替换整个alluxio工程。PS(使用alluxio-start.sh all NoMount启动集群，集群内存中数据不会丢失)。

##deploy alluxio

* 安装ssh并配置master到worker的无密登录（PS:配置ssh无密并不是alluxio本身工作需要无密登录，只是方便将文件进行远程拷贝（pssh使用），同时alluxio-start.sh all命令会用到conf下的slave host去远程启动worker进程）
* 对alluxio进行配置


先创建一个文件夹用来管理alluxio内存：
```sh
sudo mount -t ramfs -o size=100G ramfs /opt/ramfs/ramdisk
sudo chown op1:op1 /opt/ramfs/ramdisk
```

```sh
#alluxio-env.sh
export ALLUXIO_HOME=/path/to/your/alluxio
export JAVA_HOME=/path/to/your/java
export ALLUXIO_MASTER_HOSTNAME=master hostname
export ALLUXIO_LOGS_DIR=/opt/log/alluxio
export ALLUXIO_RAM_FOLDER=/opt/ramfs/ramdisk
export ALLUXIO_WORKER_MEMORY_SIZE=30GB
```

```sh
#alluxio-site.properties
alluxio.security.authorization.permission.supergroup=hadoop
alluxio.security.authentication.type=SIMPLE
alluxio.security.authorization.permission.enabled=true
alluxio.security.login.username=gl

alluxio.user.file.write.location.policy.class=alluxio.client.file.policy.RoundRobinPolicy
alluxio.keyvalue.enabled=true

alluxio.network.thrift.frame.size.bytes.max=1024MB
alluxio.user.file.readtype.default=CACHE_PROMOTE
alluxio.user.file.writetype.default=MUST_CACHE

#Tiered Storage
alluxio.worker.tieredstore.levels=2
alluxio.worker.tieredstore.level0.alias=MEM
alluxio.worker.tieredstore.level0.dirs.path=/opt/ramfs/ramdisk
alluxio.worker.tieredstore.level0.dirs.quota=30GB
alluxio.worker.tieredstore.level0.reserved.ratio=0.2
alluxio.worker.tieredstore.level1.alias=HDD
alluxio.worker.tieredstore.level1.dirs.path=/opt/data/alluxiodata
alluxio.worker.tieredstore.level1.dirs.quota=2TB
alluxio.worker.tieredstore.level1.reserved.ratio=0.1

```
如上配置已经配置了alluxio的安全性指标，如果该配置在master，那么alluxio.security.login.username指定的用户为alluxio的superuser。alluxio.user.file.write.location.policy.class 文件写block位置选择策略，使用的四种之一的RoundRobin，alluxio的块定位策略存在严重的不足
，在源码分析会进行指出，指定了user file 读写使用的策略，源码分析详细讨论，并对alluxio进行了两级存储配置，内存不足的时候数据会存放到底层慢介质，这边会有一个文件evict算法，再聊。


* 将配置好的alluxio使用scp复制到没一台worker节点

##alluxio on secure hdfs (假设已经搭建好了hadoop，本人使用cdh版本的hadoop2.6.0-cdh5.7.1)
在alluxio-env.sh加入如下配置
```sh
export ALLUXIO_UNDERFS_ADDRESS=hdfs://masterhostname:port

```
在alluxio-site.xml中加入如下配置
```sh
alluxio.master.keytab.file=/path/to/your/keytab
alluxio.master.principal=keytab content
alluxio.worker.keytab.file=/path/to/your/keytab
alluxio.worker.principal=keytab content

```

##mapreduce on alluxio 
* 在hadoop的core-site.xml中加入如下配置(每台节点)

```
<configuration>
  <property>
    <name>fs.alluxio.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
  <property>
    <name>fs.alluxio-ft.impl</name>
    <value>alluxio.hadoop.FaultTolerantFileSystem</value>
    <description>The Alluxio FileSystem (Hadoop 1.x and 2.x) with fault tolerant support</description>
  </property>
  <property>
    <name>fs.AbstractFileSystem.alluxio.impl</name>
    <value>alluxio.hadoop.AlluxioFileSystem</value>
    <description>The Alluxio AbstractFileSystem (Hadoop 2.x)</description>
  </property>
</configuration>
```
* copy alluxio-client jar到hadoop classpath中

```
cp alluxio/core/client/target/alluxio-core-client-1.4.0-jar-with-dependencies.jar /path/to/your/hadoop/lib/
```

* copy 测试数据到alluxio

```
./bin/alluxio fs copyFromLocal LICENSE /wordcount/input.txt
```

* 运行mr程序进行验证

```
bin/hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.7.1.jar wordcount alluxio://your master ip:19998/wordcount/input.txt alluxio://your master ip:19998/wordcount/output
```


##spark on alluxio
* 在spark-default.conf中加入

```
spark.executor.extraClassPath /opt/app/alluxio/core/client/target/alluxio-core-client-1.4.0-jar-with-dependencies.jar
spark.driver.extraClassPath /opt/app/alluxio/core/client/target/alluxio-core-client-1.4.0-jar-with-dependencies.jar
```

* 如果在spark-env.sh中已经定义了spark-classpath参数，那么会和上述配置冲突，解决的方式如下：在spark-shell 或spark-submit 中通过 --jars /opt/app/alluxio/core/client/target/alluxio-core-client-1.4.0-jar-with-dependencies.jar 加入jar

* spark conf中新建core-site.xml配置文件加入如下配置：

```
<configuration>
  <property>
    <name>fs.alluxio.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
  <property>
    <name>fs.alluxio-ft.impl</name>
    <value>alluxio.hadoop.FaultTolerantFileSystem</value>
    <description>The Alluxio FileSystem (Hadoop 1.x and 2.x) with fault tolerant support</description>
  </property>
  <property>
    <name>fs.AbstractFileSystem.alluxio.impl</name>
    <value>alluxio.hadoop.AlluxioFileSystem</value>
    <description>The Alluxio AbstractFileSystem (Hadoop 2.x)</description>
  </property>
</configuration>
```
* 验证spark

```
bin/spark-shell --master yarn-client --jars /opt/app/alluxio/core/client/target/alluxio-core-client-1.4.0-jar-with-dependencies.jar
val s = sc.textFile("alluxio://master ip:19998/tmp/alluxio/test.txt")
val double= s.map(line=>line+line)
double.saveAsTextFile("alluxio://master ip:19998/LICENSE2")
```


##hive on alluxio
* Hive on alluxio存在的问题很多，建议大家不要使用，等成熟时可以考虑。
* 简单说一下如何配置，首先，将alluxio的client jar复制到hive/lib下，然后在hive-metastore节点上的hive-site.xml中添加如下配置

```sh
 <property>
   <name>fs.defaultFS</name>
   <value>alluxio://your master ip:19998</value>
 </property>
 <property>
    <name>fs.alluxio.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
  <property>
    <name>fs.alluxio-ft.impl</name>
    <value>alluxio.hadoop.FaultTolerantFileSystem</value>
    <description>The Alluxio FileSystem (Hadoop 1.x and 2.x) with fault tolerant support</description>
  </property>
  <property>
    <name>fs.AbstractFileSystem.alluxio.impl</name>
    <value>alluxio.hadoop.AlluxioFileSystem</value>
    <description>The Alluxio AbstractFileSystem (Hadoop 2.x)</description>
  </property>
  <property>
    <name>alluxio.user.file.writetype.default</name>
    <value>MUST_CACHE</value>
  </property>

```

##alluxio ha

* alluxio 的ha仍然存在一些bug [JIRA](https://alluxio.atlassian.net/browse/ALLUXIO-2439?filter=-4)
* 详细配置以后给出



