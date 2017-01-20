#Alluxio RPC
分两部分：**简单介绍thrift，并结合实例讲解thrift；介绍alluixo的rpc，并通过实例讲解如何给alluxio贡献源码。**

##1.Thrift
alluxio使用thrift框架来作为整个alluxio集群的rpc通信。所以这边先简单介绍thrift，然后再通过实例来写一个thrift rpc的小demo。

###Thrift简介
Apache Thrift Thrift是Facebook于2007年开发的跨语言的rpc服框架，提供多语言的编译功能，并提供多种服务器工作模式；
用户通过Thrift的IDL（接口定义语言）来描述接口函数及数据类型，然后通过Thrift的编译环境生成各种语言类型的接口文件，
用户可以根据自己的需要采用不同的语言开发客户端代码和服务器端代码。Thrift最大的好处是实现RPC，同时支持多语言，和ProtoBuf有点类似
但是比protobuf支持的语言多，并且thrift使用起来比较简单。

###Thrift 安装
```shell
tar -xvf thrift-0.9.3.tar.gz
cd thrift-0.9.3
./configure
make 
make install
```
使用 thrift -version 来验证Thrift是否安装成功。
###Thrift demo
首先我们来写个例子，从而让大家能够直观的感受一下thrift的魅力。我们写一个简单的client向master发送算数请求，然后返回结果的一个demo。

- **1.先定义一个thrift文件**

```
namespace java com.di.thrift //thrift命令会生产 com.di.thrift.ArithmeticService的类

typedef i64 long
typedef i32 int
service ArithmeticService {
    long add(1:int num1, 2:int num2),   //方法一，并返回sum
    long multiply(1:int num1, 2:int num2), //方法二，并返回乘积
}
```
- **2.thrift --gen java arithmetic.thrift  生成thrift语言代码**

代码生产后，里面包含了Client 和Server以及Iface接口，Iface封装了service的接口，需要用户去实现。

- **3.新建一个ArithmeticServiceImpl class并继承ArithmeticService.Iface,如下：**

```java
public class ArithmeticServiceImpl implements ArithmeticService.Iface {
    public long add(int num1, int num2) throws TException {
        return num1 + num2;
    }

    public long multiply(int num1, int num2) throws TException {
        return num1 * num2;
    }
}
```
- **4.新建一个client class**

```
public class NonblockingClient {

    private void invoke() {
        TTransport transport;
        try {
            transport = new TFramedTransport(new TSocket("localhost", 7911));
            TProtocol protocol = new TBinaryProtocol(transport);

            ArithmeticService.Client client = new ArithmeticService.Client(protocol);
            transport.open();

            long addResult = client.add(100, 200);
            System.out.println("Add result: " + addResult);
            long multiplyResult = client.multiply(20, 40);
            System.out.println("Multiply result: " + multiplyResult);

            transport.close();
        } catch (TTransportException e) {
            e.printStackTrace();
        } catch (TException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        NonblockingClient c = new NonblockingClient();
        c.invoke();
    }

}

```

- **5.新建一个server class**

```
public class NonblockingServer {

    private void start() {
        try {
            TNonblockingServerTransport serverTransport = new TNonblockingServerSocket(7911);
            ArithmeticService.Processor processor = new ArithmeticService.Processor(new ArithmeticServiceImpl());

            TServer server = new TNonblockingServer(new TNonblockingServer.Args(serverTransport).
                    processor(processor));
            System.out.println("Starting server on port 7911 ...");
            server.serve();
        } catch (TTransportException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        NonblockingServer srv = new NonblockingServer();
        srv.start();
    }
}
```

ok,现在你可以启动server代码，然后再启动client代码，看看client段是不是成功的将计算结果获取到。如果想体验一把跨机器的话，可以分别将client 和server代码
打成jar包，放到不同的节点，同时将Socket hostname改成相应server机器的hostname或者ip。


###Thrift 框架
体验完一把thrift的RPC之后，我们来详细的介绍一下Thrift框架，加深大家对thrift的理解。介绍完thrift之后，我们拿alluxio的client和server
代码来进行分析讲解。

Thrift 包含一个完整的堆栈结构用于构建客户端和服务器端。下图描绘了 Thrift 的整体架构。

![架构图](Graphs/thrift-architecture.png)
如图所示是thrift的协议栈整体的架构，thrift是一个客户端和服务器端的架构体系（c/s），在最上层是用户自行实现的业务逻辑代码。 第二层是由thrift编译器自动生成的代码，
主要用于结构化数据的解析，发送和接收。TServer主要任务是高效的接受客户端请求，并将请求转发给Processor处理。Processor负责对客户端的请求做出响应，包括RPC请求转发，
调用参数解析和用户逻辑调用，返回值写回等处理。从TProtocol以下部分是thirft的传输协议和底层I/O通信。TProtocol是用于数据类型解析的，将结构化数据转化为字节流给TTransport进行传输。
TTransport是与底层数据传输密切相关的传输层，负责以字节流方式接收和发送消息体，不关注是什么数据类型。底层IO负责实际的数据传输，包括socket、文件和压缩数据流等。

**1.数据类型**
Thrift 脚本可定义的数据类型包括以下几种类型：

- 基本类型：
```
bool：布尔值，true 或 false，对应 Java 的 boolean
byte：8 位有符号整数，对应 Java 的 byte
i16：16 位有符号整数，对应 Java 的 short
i32：32 位有符号整数，对应 Java 的 int
i64：64 位有符号整数，对应 Java 的 long
double：64 位浮点数，对应 Java 的 double
string：未知编码文本或二进制字符串，对应 Java 的 String
```
- 结构体类型：
```
struct：定义公共的对象，类似于 C 语言中的结构体定义，在 Java 中是一个 JavaBean
```
- 容器类型：
```
list：对应 Java 的 ArrayList
set：对应 Java 的 HashSet
map：对应 Java 的 HashMap
```
- 异常类型：
```
exception：对应 Java 的 Exception
```
- 服务类型：
```
 service：对应服务的类
```
在上面的例子中，我们用到了一些基本的数据类型和service服务类型。thrift会根据不同的声明生成不同的code。

**2.协议层**

Thrift 可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本 (text) 和二进制 (binary) 传输协议，为节约带宽，提高传输效率，一般情况下使用二进制类型的传输协议为多数，
有时还会使用基于文本类型的协议，这需要根据项目 / 产品中的实际需求。常用协议有以下几种：

- TBinaryProtocol ： 二进制编码格式进行数据传输。用法如demo所示。
- TJSONProtocol ：使用 JSON 的数据编码协议进行数据传输。
```
//client
TJSONProtocol protocol = new TJSONProtocol(transport);
//server
TJSONProtocol.Factory proFactory = new TJSONProtocol.Factory();
```
- TSimpleJSONProtoco ：只提供 JSON 只写的协议，适用于通过脚本语言解析。
- TCompactProtocol ： 高效率的、密集的二进制编码格式进行数据传输。
```java
//用法，client端替换掉TBinaryProtocol
TCompactProtocol protocol = new TCompactProtocol(transport);
// server端
TCompactProtocol.Factory proFactory = new TCompactProtocol.Factory();
通过args.protocolFactory()将上述参数传入。
```

**3.传输层**

常用的传输层有以下几种：

- TSocket : 使用阻塞式 I/O 进行传输，是最简单的模式
- TFramedTransport ：使用非阻塞方式，按块的大小进行传输，类似于 Java 中的 NIO，如demo。
- TNonblockingTransport ：使用非阻塞方式，用于构建异步客户端
- TFileTransport：顾名思义按照文件的方式进程传输，虽然这种方式不提供Java的实现，但是实现起来非常简单。 
- TZlibTransport：使用执行zlib压缩，不提供Java的实现。

**4.服务类型**

常见的服务端类型有以下几种：

- TSimpleServer ： 单线程服务器端使用标准的阻塞式 I/O
- TThreadPoolServer ： 多线程服务器端使用标准的阻塞式 I/O
- TNonblockingServer ： 多线程服务器端使用非阻塞式 I/O

### 安全非阻塞RPC
用户先使用 keytool 生成keystore文件如：
```
keytool-genkeypair -alias certificatekey -keyalg RSA -validity 36500 -keystore .keystore
```

**client**
```
public class SecureClient {
    private void invoke() {
		TTransport transport;
		try {
			TSSLTransportFactory.TSSLTransportParameters params =
				new TSSLTransportFactory.TSSLTransportParameters();
			params.setTrustStore("tests\\src\\resouces\\truststore.jks", "111111");
			transport = TSSLTransportFactory.getClientSocket("localhost", 7911, 10000, params);
			TProtocol protocol = new TBinaryProtocol(transport);
			ArithmeticService.Client client = new ArithmeticService.Client(protocol);
			long addResult = client.add(100, 200);
			System.out.println("Add result: " + addResult);
			long multiplyResult = client.multiply(20, 40);
			System.out.println("Multiply result: " + multiplyResult);
			transport.close();
		} catch (TTransportException e) {
			e.printStackTrace();
		} catch (TException e) {
			e.printStackTrace();
		}
	}
	public static void main(String[] args) {
		SecureClient c = new SecureClient();
		c.invoke();
	}
}
```
**server**
```
public class SecureServer {
    private void start() {
		try {
			TSSLTransportFactory.TSSLTransportParameters params =
				new TSSLTransportFactory.TSSLTransportParameters();
			params.setKeyStore("tests\\src\\resouces\\keystore.jks", "111111");
			TServerSocket serverTransport = TSSLTransportFactory.getServerSocket(
				7911, 10000, InetAddress.getByName("localhost"), params);
			ArithmeticService.Processor processor = new ArithmeticService.Processor(new ArithmeticServiceImpl());

			TServer server = new TThreadPoolServer(new TThreadPoolServer.Args(serverTransport).
				processor(processor));
			System.out.println("Starting server on port 7911 ...");
			server.serve();
		} catch (TTransportException e) {
			e.printStackTrace();
		} catch (UnknownHostException e) {

		}
	}
	public static void main(String[] args) {
		SecureServer srv = new SecureServer();
		srv.start();
	}
}
```
thrift 如何集成kerberos，请参考第十章节。

##2.Alluxio rpc
alluxio使用thrift作为其RPC通信框架，下面我们通过alluxio中的一个thrift文件来进行分析。如下：

```
// core/common/src/thrift/file_system_master.thrift
namespace java alluxio.thrift

include "common.thrift"
include "exception.thrift"
struct CreateDirectoryTOptions {
  1: optional bool persisted
  2: optional bool recursive
  3: optional bool allowExists
  4: optional i16 mode
  5: optional i64 ttl
  6: optional common.TTtlAction ttlAction
}

struct CreateFileTOptions {
  1: optional i64 blockSizeBytes
  2: optional bool persisted
  3: optional bool recursive
  4: optional i64 ttl
  5: optional i16 mode
  6: optional common.TTtlAction ttlAction
}

struct FileSystemCommand {
  1: common.CommandType commandType
  2: FileSystemCommandOptions commandOptions
}

struct PersistCommandOptions {
  1: list<PersistFile> persistFiles
}

struct PersistFile {
  1: i64 fileId
  2: list<i64> blockIds
}

struct SetAttributeTOptions {
  1: optional bool pinned
  2: optional i64 ttl
  3: optional bool persisted
  4: optional string owner
  5: optional string group
  6: optional i16 mode
  7: optional bool recursive
  8: optional common.TTtlAction ttlAction
}

union FileSystemCommandOptions {
  1: optional PersistCommandOptions persistOptions
}

/**
 * This interface contains file system master service endpoints for Alluxio clients.
 */
service FileSystemMasterClientService extends common.AlluxioService {

  /**
   * Creates a directory.
   */
  void createDirectory(
    /** the path of the directory */ 1: string path,
    /** the method options */ 2: CreateDirectoryTOptions options,
    )
    throws (1: exception.AlluxioTException e, 2: exception.ThriftIOException ioe)

  /**
   * Creates a file.
   */
  void createFile(
    /** the path of the file */ 1: string path,
    /** the options for creating the file */ 2: CreateFileTOptions options,
    )
    throws (1: exception.AlluxioTException e, 2: exception.ThriftIOException ioe)

  /**
   * Frees the given file or directory from Alluxio.
   */
  void free(
    /** the path of the file or directory */ 1: string path,
    /** whether to free recursively */ 2: bool recursive,
    )
    throws (1: exception.AlluxioTException e)

  /**
   * Returns the UFS address of the root mount point.
   *
   * THIS METHOD IS DEPRECATED SINCE VERSION 1.1 AND WILL BE REMOVED IN VERSION 2.0.
   */
  string getUfsAddress() throws (1: exception.AlluxioTException e)

  /**
   * If the path points to a file, the method returns a singleton with its file information.
   * If the path points to a directory, the method returns a list with file information for the
   * directory contents.
   */
  list<FileInfo> listStatus(
    /** the path of the file or directory */ 1: string path,
    /** listStatus options */ 2: ListStatusTOptions options,
    )
    throws (1: exception.AlluxioTException e)

  /**
   * Creates a new "mount point", mounts the given UFS path in the Alluxio namespace at the given
   * path. The path should not exist and should not be nested under any existing mount point.
   */
  void mount(
    /** the path of alluxio mount point */ 1: string alluxioPath,
    /** the path of the under file system */ 2: string ufsPath,
    /** the options for creating the mount point */ 3: MountTOptions options,
    )
    throws (1: exception.AlluxioTException e, 2: exception.ThriftIOException ioe)

  /**
   * Deletes a file or a directory and returns whether the remove operation succeeded.
   * NOTE: Unfortunately, the method cannot be called "delete" as that is a reserved Thrift keyword.
   */
  void remove(
    /** the path of the file or directory */ 1: string path,
    /** whether to remove recursively */ 2: bool recursive,
    )
    throws (1: exception.AlluxioTException e)

  /**
   * Renames a file or a directory.
   */
  void rename(
    /** the path of the file or directory */ 1: string path,
    /** the desinationpath of the file */ 2: string dstPath,
    )
    throws (1: exception.AlluxioTException e, 2: exception.ThriftIOException ioe)

  /**
   * Sets file or directory attributes.
   */
  void setAttribute(
    /** the path of the file or directory */ 1: string path,
    /** the method options */ 2: SetAttributeTOptions options,
    )
    throws (1: exception.AlluxioTException e)

}

```
对于上述thrift语言文件，我删除了一部分不然篇幅就显得太长了。可以清楚的看到，struct定义了很多文件操作相对于的操作参数。而service定义了很多文件操作接口，并将struct作为操作参数。

- **这里有需要注意的地方，如果大家修改了thrift文件，不需要使用thrift命令来生产thrift java代码，alluxio提供了脚本来生产，alluxio thriftGen就可以自动生成修改过得thrift文件**


接下来通过client端创建一个文件夹为例来说明alluxio如何进行RPC通信的。
```
alluxio fs mkdir /your/alluxio/path
``` 
### shell
找到MkdirCommand.java文件: /shell/src/main/java/shell/command/MkdirCommand.java

```java
  public void run(CommandLine cl) throws AlluxioException, IOException {
    String[] args = cl.getArgs();
    for (String path : args) {
      AlluxioURI inputPath = new AlluxioURI(path);

      CreateDirectoryOptions options = CreateDirectoryOptions.defaults().setRecursive(true);
      mFileSystem.createDirectory(inputPath, options);
      System.out.println("Successfully created directory " + inputPath);
    }
  }
```
代码很简单，首先new一个CreateDirectoryOptions的类。
然后调用client端的createDirectory方法。
- 这边注意一点CreateDirectoryOptions是client端自己写的一个类，它和thrift文件中定义的类不一样CreateDirectoryTOptions,所以自己实现选项类的时候需要加一个toThrift方法，将自己的参数转换为thrift API能够接受的参数。

下面来看看client端的代码和server端的代码：
###client
找到BaseFileSystem 类中的createDirectory方法。
``` java
  @Override
  public void createDirectory(AlluxioURI path, CreateDirectoryOptions options)
      throws FileAlreadyExistsException, InvalidPathException, IOException, AlluxioException {
    FileSystemMasterClient masterClient = mFileSystemContext.acquireMasterClient();
    try {
      masterClient.createDirectory(path, options);
      LOG.debug("Created directory " + path.getPath());
    } finally {
      mFileSystemContext.releaseMasterClient(masterClient);
    }
  }
```
接着调用FileSystemMasterClient的createDirectory
```
  public synchronized void createDirectory(final AlluxioURI path,
      final CreateDirectoryOptions options) throws IOException, AlluxioException {
    retryRPC(new RpcCallableThrowsAlluxioTException<Void>() {
      @Override
      public Void call() throws AlluxioTException, TException {
        mClient.createDirectory(path.getPath(), options.toThrift());
        return null;
      }
    });
  }
```
看到这里，才是真正意义上的调用thrift生成的client代码，它接收的参数options必须转换成带T的类。
```
mClient.createDirectory(path.getPath(), options.toThrift());
```
mClient: FileSystemMasterClientService 类是我们定义的thrift文件生成的class，里面的Iface是必须要实现的。可以找到master端的实现代码。


###server
FileSystemMasterClientServiceHandler类为IFace的实现代码，但是真正操作元数据的是FileSystemMaster这个类。

```
  public void createDirectory(final String path, final CreateDirectoryTOptions options)
      throws AlluxioTException, ThriftIOException {
    RpcUtils.call(new RpcCallableThrowsIOException<Void>() {
      @Override
      public Void call() throws AlluxioException, IOException {
        mFileSystemMaster.createDirectory(new AlluxioURI(path),
            new CreateDirectoryOptions(options));
        return null;
      }
    });
  }
```
如FileSystemMasterClientServiceHandler类中createDirectory方法，调用了FileSystemMaster的createDirectory方法

```
  public void createDirectory(AlluxioURI path, CreateDirectoryOptions options)
      throws InvalidPathException, FileAlreadyExistsException, IOException, AccessControlException,
      FileDoesNotExistException {
    LOG.debug("createDirectory {} ", path);
    Metrics.CREATE_DIRECTORIES_OPS.inc();
    long flushCounter = AsyncJournalWriter.INVALID_FLUSH_COUNTER;
    try (LockedInodePath inodePath = mInodeTree.lockInodePath(path, InodeTree.LockMode.WRITE)) {
      mPermissionChecker.checkParentPermission(Mode.Bits.WRITE, inodePath);
      mMountTable.checkUnderWritableMountPoint(path);
      flushCounter = createDirectoryAndJournal(inodePath, options);
    } finally {
      // finally runs after resources are closed (unlocked).
      waitForJournalFlush(flushCounter);
    }
  }
```
如上，先对path进行WRITE加锁，然后再对该路径进行权限认证，如果没有create的权限的话，会直接报AccessControlException异常的。

- **alluxio目前提供的权限认证还是太水，用户通过设置username就可以修改成相应用户，从而获得该用户的权限**

最后，会将dir的元数据加入到InodeTree中。接下来看看InodeTree数据结构。

### InodeTree
alluxio自己实现了一种数据结构来存储文件元数据。
```
  public void initializeRoot(Permission permission) {
    if (mRoot == null) {
      mRoot = InodeDirectory
          .create(mDirectoryIdGenerator.getNewDirectoryId(), NO_PARENT, ROOT_INODE_NAME,
              CreateDirectoryOptions.defaults().setPermission(permission));
      mRoot.setPersistenceState(PersistenceState.PERSISTED);
      mInodes.add(mRoot);
      mCachedInode = mRoot;
    }
  }
```
所有的节点都放到了mInodes里面,mInodes数据结构是：
```
  private final FieldIndex<Inode<?>> mInodes = new UniqueFieldIndex<>(ID_INDEX);
```
下面看看UniqueFieldIndex类。
```
public class UniqueFieldIndex<T> implements FieldIndex<T> {
  private final IndexDefinition<T> mIndexDefinition;
  private final ConcurrentHashMapV8<Object, T> mIndexMap;

  /**
   * Constructs a new {@link UniqueFieldIndex} instance.
   *
   * @param indexDefinition definition of index
   */
  public UniqueFieldIndex(IndexDefinition<T> indexDefinition) {
    mIndexMap = new ConcurrentHashMapV8<>(8, 0.95f, 8);
    mIndexDefinition = indexDefinition;
  }

  @Override
  public boolean add(T object) {
    Object fieldValue = mIndexDefinition.getFieldValue(object);
    T previousObject = mIndexMap.putIfAbsent(fieldValue, object);

    if (previousObject != null && previousObject != object) {
      return false;
    }
    return true;
  }

  @Override
  public boolean remove(T object) {
    Object fieldValue = mIndexDefinition.getFieldValue(object);
    return mIndexMap.remove(fieldValue, object);
  }

  @Override
  public void clear() {
    mIndexMap.clear();
  }

  @Override
  public boolean containsField(Object fieldValue) {
    return mIndexMap.containsKey(fieldValue);
  }

  @Override
  public boolean containsObject(T object) {
    Object fieldValue = mIndexDefinition.getFieldValue(object);
    T res = mIndexMap.get(fieldValue);
    if (res == null) {
      return false;
    }
    return res == object;
  }

  @Override
  public Set<T> getByField(Object value) {
    T res = mIndexMap.get(value);
    if (res != null) {
      return Collections.singleton(res);
    }
    return Collections.emptySet();
  }

  @Override
  public T getFirst(Object value) {
    return mIndexMap.get(value);
  }

  @Override
  public Iterator<T> iterator() {
    return mIndexMap.values().iterator();
  }

  @Override
  public int size() {
    return mIndexMap.size();
  }
}

```
其中由ConcurrentHashMapV8 和IndexDefinition来维护元数据，ConcurrentHashMapV8是netty实现的java8 的concurrentHashMap。 IndexDefinition用来维护id的唯一性。

add的时候，首先获取object的id，然后id作为key，put到hashmap中。同时在插入的过程中，会将path转换成Inode，Inode分为两种：InodeFile和InodeDirectory，InodeFile会有指向parent的引用，一样，InodeDirectory
会用child数组来记录自己的孩子节点。


