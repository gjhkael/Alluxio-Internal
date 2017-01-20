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

##2.Alluxio rpc

###client


###server
