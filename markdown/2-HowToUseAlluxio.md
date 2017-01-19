#How to use Alluxio

##Alluxio Command

- alluxio protoGen 当用户在修改了alluxio的protobuf文件的时候，需要使用这个命令来重新生成protobuf序列化程序
- alluxio thriftGen 当用户修改了alluxio的thrift文件的时候，需要只用这个命令来重新生成PPC java程序
- alluxio fs 用来操作alluxio中的文件
- alluxio fs mount /allxuio/path hdfs://xxx/path 
- alluxio fs setTtl 设置文件的ttl，目前1.4还不支持文件夹的ttl设置，master已经merge了我写的这个patch，可以将master中[PR（click here）](https://github.com/Alluxio/alluxio/pull/4458) cherry-pick到自己的alluxio源码中
- alluxio fs checkConsistency 可以查看底层文件系统和alluxio是否存在文件不一致问题。但是当前版本1.4.0没有提供解决一致性的命令。可以参考我提
的PR[ Make consistency ](https://github.com/Alluxio/alluxio/pull/4686)。
- 其他的操作可以使用 alluxio fs来看具体的含义

##Alluxio API
alluxio包装了hdfs的api，所以可以使用两种方式来读写alluxio文件：
###alluxio
```java
public Boolean call() throws Exception {
    	Configuration.set(PropertyKey.MASTER_HOSTNAME, "10.2.4.192");
		Configuration.set(PropertyKey.MASTER_RPC_PORT, Integer.toString(19998));
		FileSystem fs = FileSystem.Factory.get();
		Configuration.set(PropertyKey.SECURITY_LOGIN_USERNAME, "hdfs");
		System.out.println(Configuration.get(PropertyKey.MASTER_ADDRESS));
		writeFile(fs);
		return readFile(fs);
		//writeFileWithAbstractFileSystem();
		//return true;

	}

	private void writeFile(FileSystem fileSystem)
		throws IOException, AlluxioException {
		ByteBuffer buf = ByteBuffer.allocate(NUMBERS * 4);
		buf.order(ByteOrder.nativeOrder());
		for (int k = 0; k < NUMBERS; k++) {
			buf.putInt(k);
		}
		LOG.debug("Writing data...");
		long startTimeMs = CommonUtils.getCurrentMs();
		if (fileSystem.exists(mFilePath)) {
			fileSystem.delete(mFilePath);
		}
		FileOutStream os = fileSystem.createFile(mFilePath, mWriteOptions);
		os.write(buf.array());
		os.close();
		System.out.println((FormatUtils.formatTimeTakenMs(startTimeMs, "writeFile to file " + mFilePath)));
	}

	private boolean readFile(FileSystem fileSystem)
		throws IOException, AlluxioException {
		boolean pass = true;
		LOG.debug("Reading data...");
		final long startTimeMs = CommonUtils.getCurrentMs();
		FileInStream is = fileSystem.openFile(mFilePath, mReadOptions);
		ByteBuffer buf = ByteBuffer.allocate((int) is.remaining());
		is.read(buf.array());
		buf.order(ByteOrder.nativeOrder());
		for (int k = 0; k < NUMBERS; k++) {
			pass = pass && (buf.getInt() == k);
			System.out.print(pass);
		}
		is.close();
		System.out.println(FormatUtils.formatTimeTakenMs(startTimeMs, "readFile file " + mFilePath));
		return pass;
	}
```

###hadoop
```java
public void writeFileWithAbstractFileSystem() {
    	org.apache.hadoop.conf.Configuration conf = new org.apache.hadoop.conf.Configuration();
		conf.set("alluxio.master.hostname", "10.2.4.192");
		conf.set("alluxio.master.port", "19998");
		conf.set("alluxio.security.login.username", "gl");
		conf.set("fs.alluxio.impl", "alluxio.hadoop.FileSystem");
		conf.set("alluxio.user.file.writetype.default", "CACHE_THROUGH");
		try {
			Path p = new Path("alluxio://10.2.4.192:19998/tmp/test4");
			org.apache.hadoop.fs.FileSystem fileSystem = p.getFileSystem(conf);
			

			System.out.println(System.currentTimeMillis());
			fileSystem.mkdirs(p);
			System.out.println(System.currentTimeMillis());

		} catch (Exception e) {
			e.printStackTrace();
		}
	}

```

##Alluxio KV
alluxio支持kv数据类型的写，但是alluxio并不支持文件的追加写，所以kv数据类型并不支持文件追加写，同时要求k是排好序，并且不重复。可以说alluxio
的kv使用场景十分有限。

```java 
  // alluxio.hadoop.AbstractFileSystem
  public FSDataOutputStream append(Path path, int bufferSize, Progressable progress)
      throws IOException {
    LOG.debug("append({}, {}, {})", path, bufferSize, progress);
    if (mStatistics != null) {
      mStatistics.incrementWriteOps(1);
    }
    AlluxioURI uri = new AlluxioURI(HadoopUtils.getPathWithoutScheme(path));
    try {
      if (mFileSystem.exists(uri)) {
        throw new IOException(ExceptionMessage.FILE_ALREADY_EXISTS.getMessage(uri));
      }
      return new FSDataOutputStream(mFileSystem.createFile(uri), mStatistics);
    } catch (AlluxioException e) {
      throw new IOException(e);
    }
  }

```
下面来看个demo：
```java
public static void main(String[] args) throws Exception {
    if (args.length != 1) {
      System.out.println("You must give me the file path");
      System.exit(-1);
    }
    Configuration.set(PropertyKey.MASTER_HOSTNAME,"you hostname");
    AlluxioURI storeUri = new AlluxioURI(args[0]);
    KeyValueSystem kvs = KeyValueSystem.Factory.create();

    // Creates a store.
    KeyValueStoreWriter writer = kvs.createStore(storeUri);

    // Puts a key-value pair ("key", "value").
    String key = "key";
    String value = "value";
    writer.put(key.getBytes(), value.getBytes());
    System.out.println(String.format("(%s, %s) is put into the key-value store", key, value));

    // Completes the store.
    writer.close();

    // Opens a store.
    KeyValueStoreReader reader = kvs.openStore(storeUri);

    // Gets the value for "key".
    System.out.println(String.format("Value for key '%s' got from the store is '%s'", key,
        new String(reader.get(key.getBytes()))));

    // Closes the reader.
    reader.close();
    //接下来的操作会导致文件已存在的异常。
    AlluxioURI storeUri = new AlluxioURI(args[0]);
    KeyValueSystem kvs = KeyValueSystem.Factory.create();
    // Creates a store.
    KeyValueStoreWriter writer = kvs.createStore(storeUri);
    // Puts a key-value pair ("key1", "value1").
    String key = "key1";
    String value = "value1";
    writer.put(key.getBytes(), value.getBytes());
  }

```


##Alluxio Write Type
alluxio.user.file.writetype.default 配置用来设置alluxio write的类型，总共5种：
###MUST_CACHE
如果用户使用MUST_CACHE，文件只会在alluxio的内存层，并不会发生持久化的过程。这种写方式是效率最高的，但是数据的容错性就没办法保证。
alluxio没有多副本机制，所以worker crash会导致数据的丢失。
###TRY_CACHE
在最新的版本中以及放弃了这种写类型。
###CACHE_THROUGH
数据写alluxio的时候，同步到底层文件系统，我们知道数据写到alluxio的时候同时又要再写到底层文件系统，效率肯定会比直接写hdfs差。所以
这边用户在选择这个写类型的时候要有所考量。
###THROUGH
直接跳过alluxio，直接写到底层文件系统。这种方式和直接写hdfs一样。
###ASYNC_THROUGH
数据写alluxio的时候，异步写到底层文件系统，这种方式仍然存在丢失数据的风险。

##Alluxio Read Type
alluxio文件读有3中策略。分别是：NO_CACHE、CACHE、CACHE_PROMOTE

###NO_CACHE
如果用户使用了alluxio.user.file.readtype.default=NO_CACHE 来指定alluxio的读类型，那么alluxio会跨过alluxio内存，直接从底层文件系统读取数据。

###CACHE
该策略会将底层文件系统的数据加载到alluxio存储层，但是alluxio多级存储的情况不会发生数据跨级移动。也就是说，直接从文件所属层访问数据

###CACHE_PROMOTE
该策略会读数据并且将底层文件系统的数据写入alluxio内存层，同时内存不够的时候会发生文件的置换。

##Alluxio FileWritePolicy
alluxio 总共有4种文件write，block位置选择策略。文件在写入到alluxio分布式内存文件系统时，和HDFS一样，都是以Block的形式来存储数据，因此Block的位置
可以以不同的方式来决定存放在那台机器上。

###LocalFirstPolicy
不多说，直接上源码，每个文件写入的的时候，都会调用getWorkerForNextBlock方法来获取下一个block的存放地址，返回hostIp.
该策略是优先选择local，当local的CapacityBytes（容量）不满足block的时候，会选择其他worker。
```java
  // alluxio.client.file.policy.LocalFirstPolicy
  public WorkerNetAddress getWorkerForNextBlock(Iterable<BlockWorkerInfo> workerInfoList,
      long blockSizeBytes) {
    // first try the local host
    for (BlockWorkerInfo workerInfo : workerInfoList) {
      if (workerInfo.getNetAddress().getHost().equals(mLocalHostName)
          && workerInfo.getCapacityBytes() >= blockSizeBytes) {
        return workerInfo.getNetAddress();
      }
    }

    // otherwise randomly pick a worker that has enough availability
    List<BlockWorkerInfo> shuffledWorkers = Lists.newArrayList(workerInfoList);
    Collections.shuffle(shuffledWorkers);
    for (BlockWorkerInfo workerInfo : workerInfoList) {
      if (workerInfo.getCapacityBytes() >= blockSizeBytes) {
        return workerInfo.getNetAddress();
      }
    }
    return null;
  }

```
注意点：

- 1.上述判断条件是CapacityBytes，而不是avalibaleBytes，所以当block size小于CapacityBytes的时候，不管当前worker是否还有空间，都会返回
当前local。当空间不够的时候会发生文件置换的现象，使用的策略是LRU。

- 2.workerInfo中的UsedBytes并不是实时更新，只有当文件写完后才会更新这个变量。所以我提了一个[PR](https://github.com/Alluxio/alluxio/pull/4445) 来解决这个问题，但是因为UsedBytes不是
实时更新，所以需要配置其他参数来预留足够的空间来存储当前文件。

###MostAvailableFirstPolicy
代码很简单，只是每次获取block的地址时，使用最大容量减去使用容量剩下最多的woker。
```java
  // alluxio.client.file.policy.LocalFirstPolicy
  public WorkerNetAddress getWorkerForNextBlock(Iterable<BlockWorkerInfo> workerInfoList,
      long blockSizeBytes) {
    long mostAvailableBytes = -1;
    WorkerNetAddress result = null;
    for (BlockWorkerInfo workerInfo : workerInfoList) {
      if (workerInfo.getCapacityBytes() - workerInfo.getUsedBytes() > mostAvailableBytes) {
        mostAvailableBytes = workerInfo.getCapacityBytes() - workerInfo.getUsedBytes();
        result = workerInfo.getNetAddress();
      }
    }
    return result;
  }
```
注意点：

- 如果数据在一台节点上，使用这种方式，无疑会增加系统的写延迟，因为需要走网络，而LocalFirst则只会发生内存的拷贝，相对来说更加的高效，
所以如果有计算引擎架构在alluxio之上的时候，因为本身数据较为均匀的分布在各个节点，所以LocalFist要比其他策略高效。


###RoundRobinPolicy
```java
 // alluxio.client.file.policy.RoundRobinPolicy
  public WorkerNetAddress getWorkerForNextBlock(Iterable<BlockWorkerInfo> workerInfoList,
      long blockSizeBytes) {
    if (!mInitialized) {
      mWorkerInfoList = Lists.newArrayList(workerInfoList);
      Collections.shuffle(mWorkerInfoList);
      mIndex = 0;
      mInitialized = true;
    }

    // at most try all the workers
    for (int i = 0; i < mWorkerInfoList.size(); i++) {
      WorkerNetAddress candidate = mWorkerInfoList.get(mIndex).getNetAddress();
      BlockWorkerInfo workerInfo = findBlockWorkerInfo(workerInfoList, candidate);
      mIndex = (mIndex + 1) % mWorkerInfoList.size();
      if (workerInfo != null && workerInfo.getCapacityBytes() >= blockSizeBytes) {
        return candidate;
      }
    }
    return null;
  }
```
很简单，每次都是随机选择一个满足条件的worker

###SpecificHostPolicy
```java
  // alluxio.client.file.policy.SpecificHostPolicy
  public WorkerNetAddress getWorkerForNextBlock(Iterable<BlockWorkerInfo> workerInfoList,
      long blockSizeBytes) {
    // find the first worker matching the host name
    for (BlockWorkerInfo info : workerInfoList) {
      if (info.getNetAddress().getHost().equals(mHostname)) {
        return info.getNetAddress();
      }
    }
    return null;
  }

```

##总结

总的来说，alluxio的使用方面还是比较简单，如果熟悉HDFS的，很快就能熟练Alluxio的使用。

