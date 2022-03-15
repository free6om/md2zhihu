通过[前文](https://zhuanlan.zhihu.com/p/457450880)的介绍，大家基本了解了一条SQL在galaxysql中的解析和执行流程，由于galaxysql是无状态的计算节点，真正的数据还需要从存储节点传输到计算节点上，这部分工作则是由私有协议完成的。本文将从传输到数据节点的请求开始，到数据返回给计算节点结束，着眼于私有协议连接的完整生命周期，介绍私有协议的关键代码。

# 概述

为了充分发挥数据节点的本地计算能力，同时尽可能减少网络数据传输量，计算节点会将尽可能多的计算内容下推，因此对单个存储节点的数据请求可能是一个非常复杂的join查询，也可能是一个非常简单的索引点查。同时由于一个逻辑表存在多个物理分片，计算节点和存储节点的请求会话的数量会随着分片数的增多而成倍放大，传统的MySQL协议+连接池的架构已经不能满足PolarDB-X的需求，私有协议就是在这种需求场景下应运而生的。

如下图所示，私有协议采用了连接与会话分离的RPC协议设计理念，在同个TCP通道中支持多个会话，同时分别具备流控机制，全双工响应式的工作模式，允许请求流水线，具备高吞吐、可扩展等多种特性。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645516880237-9f22936a-0fd3-4e5c-9c86-be6658bf803d.png)

更多关于私有协议解决上述问题的设计，可以参考之前文章[PolarDB-X私有协议设计](https://zhuanlan.zhihu.com/p/308173106)，本文主要从代码角度详细描述下私有协议的工作流程。

我们将从计算节点和存储节点两个部分完整梳理下私有协议连接的一生。由于篇幅限制，本文只涉及计算节点上私有协议的处理，存储节点上的私有协议留在私有协议连接的一生（DN篇）中再说明。

# 计算节点

计算节点在私有协议中担任的角色是客户端，负责发送下推的请求，同时接收返回的数据。

## 网络层框架

谈到网络通信协议设计和实现，网络层框架的设计的必不可少的，为了追求极致的性能，PolarDB-X私有协议的网络层没有采用现有的网络库，而是使用java的NIO实现的一套精简的定制化Reactor框架。这部分代码改进自[galaxysql](https://github.com/ApsaraDB/galaxysql)中的Reactor框架，网络层初始化在[NIOWorker](https://github.com/ApsaraDB/galaxyglue/blob/f6d5e2c16a8253df356a00f55fb61bc7b1f08e57/src/main/java/com/alibaba/polardbx/rpc/net/NIOWorker.java#L55)中，初始化CPU core数2倍（最大限制为32）的[NIOProcesser](https://github.com/ApsaraDB/galaxyglue/blob/f6d5e2c16a8253df356a00f55fb61bc7b1f08e57/src/main/java/com/alibaba/polardbx/rpc/net/NIOProcessor.java#L27)，而NIOProcesser是[NIOReactor](https://github.com/ApsaraDB/galaxyglue/blob/f6d5e2c16a8253df356a00f55fb61bc7b1f08e57/src/main/java/com/alibaba/polardbx/rpc/net/NIOReactor.java#L33)的包裹，后者是Reactor框架的具体实现，每个Reactor使用独立的堆外内存池作为收发包的缓冲，总缓冲内存大小限制为堆内存大小的10%。

NIO收到的包会直接通过回调函数调用到注册处理函数上，而发送的数据会在调用时候仅写入到send buf中，而网络写入则是有单独的一个[线程](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOReactor.java#L140)去完成，在flush的时候会显式地触发一次事件唤醒该线程，写线程优先写入TCP send buf，当写不下时，会注册OP_WRITE事件，等待可写后再写入剩下内容。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645516825727-30417bfe-7632-4e7a-a514-658af2d9044b.png)

数据包的编码和解码则是在[NIOClient](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOClient.java)中实现的。为了实现最佳的性能，[解包流程](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOClient.java#L296)直接在堆外内存上进行，使用protobuf对流直接解析，将解包的结果放到堆内。堆外的内存被切分成若干64KB的chunk，每个Reactor会独占一个chunk作为接收缓冲，并且在其上进行连续解析和复用，利用CPU cache最大化接收、解析效率。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645517487852-d964a706-a432-4953-8a8d-0e9078eb0fe4.png)

而对于超出chunk大小的特大包，会额外构造一个堆内大buffer，用于接收和解析，而超大包的回退flag会在定时探活任务中重置，在连续10s中没有超大包出现的情况下，会释放掉这个堆内内存，回退到高性能的堆外64KB buffer上进行接收和解码。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645517824120-30f51255-99f9-4ccc-ab20-b0711d3cfb7b.png)

请求的发送也深入集成到了[NIOClient](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOClient.java#L529)之中，writer会优先尝试写入到发送缓冲队列队尾的buffer中，如果容量不足，则会新申请个buffer然后进行填充，链到队尾。这里的buffer也是从之前给每个Reactor预分配的堆外缓冲池中拿的，当发送的包超过chunk大小，也会分配对应的堆内buf用于请求的序列化。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645518007594-b7d32abe-f0d1-4a2d-84a7-3b66a0d23794.png)

同时NIOClient也负责TCP连接的[建立](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOClient.java#L145)和[断连资源释放](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/net/NIOClient.java#L267)，作为底层网络资源管理的完全独立实现。请求和数据的包各个字段的定义可以参考[proto](https://github.com/ApsaraDB/galaxyglue/tree/main/src/main/proto)这里就不再展开。

## 连接及会话

梳理完了网络层，下面来到连接与会话分离的具体实现了，因为剥离了连接及收发包的具体实现，连接和会话的管理就变得清晰和简洁许多了。

首先是一个TCP连接的逻辑抽象结构，这里我们是在[XClient](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XClient.java)中实现的，之所以的取名为client是为了和JDBC模型中的Connection区别开来，避免误解。该类主要管理一个TCP连接已经上面并行跑着的会话，负责TCP完整生命周期的管理，认证鉴权，同时也会维护些公共信息。

其中最重要的成员变量则是[workingSessionMap](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XClient.java#L74)记录了该TCP连接上并行运行的所有会话映射关系，可以快速地通过会话ID找到对应的会话抽象结构[XSession](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java)。

XSession中则是提供了所有和会话相关的请求函数和相关的信息存储，包括[执行计划的请求](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1547)、[SQL Qeury请求](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1380)、[SQL Update请求](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1547)、[TSO请求](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1708)、[Session变量处理](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L607)、[数据包处理及异步唤醒](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1049)等诸多处理函数。

## 连接池及全局单例管理器

为了达到更好的性能，TCP连接和会话的复用也是必不可少的，这里由于连接和会话的解绑，连接池不仅仅是缓存了到计算节点的TCP连接，也缓存了到计算节点的会话。

[XClientPool](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XClientPool.java)是对到一个存储节点的连接池管理结构，其中目标存储节点由【IP，端口，用户名】这三元组唯一确定，同时该类还存储了到这个目标存储节点的全部TCP连接（即XClient）和全部建立了的会话（即XSession）。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645522734105-240f77da-5d9e-4afa-b4df-3e6765be30fe.png)

XClientPool中实现了存储节点的会话获取，即对应JDBC接口中的getConnection，同时也实现了针对该存储节点所有连接和会话的生命周期管理、连接探活、会话预分配等功能。

实现了单个存储节点的连接池之后，我们需要一个全局单例管理所有连接池以及调度私有协议相关的定时任务，这个就是[XConnectionManager](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XConnectionManager.java)的工作了，XConnectionManager维护了一个目标存储节点三元组到实例连接池的映射，同时维护了一个定时任务线程池，实现定时探活、会话&连接最长生命控制以及连接池预热等功能。

## JDBC兼容层

一个新的SQL协议层对对上层使用者的要求是比较高的，为了提升开发效率，私有协议提供了兼容JDBC的使用方法，可以在上层调用不用过多改动的情况下，平滑地从JDBC切换到私有协议，同时也提供了协议热切换的能力。

JDBC兼容层代码目录在[compatible](https://github.com/ApsaraDB/galaxyglue/tree/main/src/main/java/com/alibaba/polardbx/rpc/compatible)目录下，Connection的继承因为历史原因，文件在[XConnection](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XConnection.java)这里。JDBC兼容层提供了包括[DataSource](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/compatible/XDataSource.java)、[Connection](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XConnection.java)、[Statement](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/compatible/XStatement.java)、[PreparedStatemet](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/compatible/XPreparedStatement.java)、[ResultSet](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/compatible/XResultSet.java)、[ResultSetMetaData](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/compatible/XResultSetMetaData.java)在内的大多数常用接口函数实现，不支持的函数都会明确抛出异常避免误用。

## 整体关系

至此私有协议计算节点端的大部分结构都已说明完成，下面给出一个整体的关系图。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645670135043-c03b09e8-e436-4aef-961f-cf686fd44053.png)

# 私有协议连接的一生（CN视角）

简单了解了私有协议的各层实现后，我们以一条发到存储节点的请求为例，完整梳理下执行的流程。这里我们绕开计算节点的复杂流程，直接以下面代码为例（注：因为是绕开计算节点的启动，需要将[com.alibaba.polardbx.rpc.XConfig#GALAXY_X_PROTOCOL](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/XConfig.java#L71)手动设置为true）。

```java
public class GalaxyTest {
    public final static String SERVER_IP = "127.0.0.1";
    public final static int SERVER_PORT = 31306;
    public final static String SERVER_USR = "root";
    public final static String SERVER_PSW = "root";
    private final static String DATABASE = "test";

    static XDataSource dataSource = new XDataSource(SERVER_IP, SERVER_PORT, SERVER_USR, SERVER_PSW, DATABASE, null);

    public static XConnection getConn() throws Exception {
        return (XConnection) dataSource.getConnection();
    }

    public static List<List<Object>> getResult(XResult result) throws Exception {
        return getResult(result, false);
    }

    public static List<List<Object>> getResult(XResult result, boolean stringOrBytes) throws Exception {
        final List<PolarxResultset.ColumnMetaData> metaData = result.getMetaData();
        final List<List<Object>> ret = new ArrayList<>();
        while (result.next() != null) {
            final List<ByteString> data = result.current().getRow();
            assert metaData.size() == data.size();
            final List<Object> row = new ArrayList<>();
            for (int i = 0; i < metaData.size(); ++i) {
                final Pair<Object, byte[]> pair = XResultUtil
                    .resultToObject(metaData.get(i), data.get(i), true,
                        result.getSession().getDefaultTimezone());
                final Object obj =
                    stringOrBytes ? (pair.getKey() instanceof byte[] || null == pair.getValue() ? pair.getKey() :
                        new String(pair.getValue())) : pair.getKey();
                row.add(obj);
            }
            ret.add(row);
        }
        return ret;
    }

    private void show(XResult result) throws Exception {
        List<PolarxResultset.ColumnMetaData> metaData = result.getMetaData();
        for (PolarxResultset.ColumnMetaData meta : metaData) {
            System.out.print(meta.getName().toStringUtf8() + "\t");
        }
        System.out.println();
        final List<List<Object>> objs = getResult(result);
        for (List<Object> list : objs) {
            for (Object obj : list) {
                System.out.print(obj + "\t");
            }
            System.out.println();
        }
        System.out.println("" + result.getRowsAffected() + " rows affected.");
    }

    @Ignore
    @Test
    public void playground() throws Exception {
        try (XConnection conn = getConn()) {
            conn.setStreamMode(true);
            final XResult result = conn.execQuery("select 1");
            show(result);
        }
    }
}
```

直接运行playground可以看到预期的select 1的结果，下面我们就这段代码深入跟踪说明。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645587450895-ed59eaf5-d15a-4567-ad7c-99c4843297f9.png)

## 数据源初始化

要使用私有协议，需要先new一个对应存储节点的XDataSource，XDataSource构造过程中，会到XConnectionManager中注册一个新的实例连接池，如果对应连接池已存在，则会将已有连接池的引用计数加一。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645587869688-9128df31-61af-4590-8486-6b360f220260.png)

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645587925152-840e28f5-a57c-4550-a9cd-64ac403cd2c4.png)

## 获取Connection

当需要到存储节点上执行查询时，首先是需要获取一个会话，无论是显式开启事务还是使用auto commit事务，会话都是执行这些请求的最小上下文，在JDBC的模型中对应的即是getConnection，这里我们通过XDataSource的getConnection方法便可以拿到一个到对应存储节点的会话。首先XDataSource会根据存储的【IP，端口，用户名】这三元组查找到XConnectionManager中的连接池，在通过最高并发检查后，会话的获取逻辑在[XClientPool](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XClientPool.java#L281)中实现。首先会尝试在空闲会话池中拿会话，在通过重置检查和初始化后会返回给调用者。大部分场景都会走到这条路径，ConcurrentLinkedQueue也提供了较好的并发性能。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645605837489-849f7aa8-ee55-4f17-b6fa-b861b2c8afe2.png) 
在我们这个代码的场景下，由于数据源刚新建，后台的定时任务还没跑过，所以idleSessions为空，会进入到下面流程中，尝试找到已有的TCP连接，并选择合适的连接并在其上建立新的会话。具体的策略是，优先选择没有会话的TCP连接进行会话创建，其次在TCP连接未达到上限的情况下，优先创建TCP连接，当连接达到上限后，round robin策略在TCP连接上进行复用会话创建。即总的策略是优先一连接一会话，只有当会话数超过连接数上限后，才开始多会话复用。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645606546282-725a3ef0-3485-4b0e-bfaa-887fbf8aa009.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645606581429-90a47870-bf2d-4d60-9d1c-3fccd60c5e36.png) 
同样，当前代码场景下，我们也没有创建好的TCP连接，流程进入到最后的连接创建流程，这里会有一把大锁锁住连接池，在TCP连接未达上限且没有超时的情况下，快速新建一个XClient占坑。而如果超限了，则会sleep 10ms进入busy waiting循环。真正的TCP connect（waitChannel）会在锁外被调用，首先client会以阻塞模式带超时方式connect，然后切换为非阻塞模式，round robin策略注册到一个NIOProcesser上，在返回时，该TCP连接已经成功建立。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645607099835-1dd473ee-a0cb-4b12-a631-5ace86a8e861.png) 
为了兼顾安全和性能，连接鉴权在TCP建连后只用做一次，而会话创建不需要鉴权。鉴权是在[initClient](https://github.com/ApsaraDB/galaxyglue/blob/f6d5e2c16a8253df356a00f55fb61bc7b1f08e57/src/main/java/com/alibaba/polardbx/rpc/client/XClient.java#L607)中完成。这里我们只会发一个SESS_AUTHENTICATE_START_VALUE的包，后续校验则由回调完成。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645608422756-6c97e57c-3519-4bf0-9208-39e4e6da10e5.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645608636043-7deaf7cc-5b25-414c-a4c1-fbc1db63022d.png) 
认证采用标准的MySQL41认证流程，server端会返回一个challenge值，将库名，用户名和加盐hash后的密码返回给MySQL即可完成认证。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645608827235-61893852-cc1d-431d-8d6c-a91818cf7758.png) 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645608987069-45287b74-0b8b-4364-95ff-1d24fd4ce329.png)

至此，我们到存储节点的TCP连接就已经建好了，下面就是创建会话了，其实创建会话是一个异步的流程，早在我们创建新XClient的时候，XConnection就已经new好了，在[这里](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/pool/XClientPool.java#L392)下断点跟进去即可看到newXSession的流程，其本质就是分配了一个session id，并把其状态初始化为init，最后把XSession绑定到一个XConnection上。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645609271924-0e83895b-3164-4411-bb86-3411b05fe20b.png) 
最后XConnection经过初始化（重置auto commit状态），重置默认DB、默认字符集（这两个都是lazy操作），记录一些统计信息，就返回给用户使用了。

## 发送查询请求

现在我们拿到了一个初始化好的兼容JDBC的Connection，为了简化流程，这里我们直接调用了XConnection中的execQuery，这个函数等价于直接创建一个Statement然后执行。XConnection的execQuery是XSession中[execQuery](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/client/XSession.java#L1380)的包装，这里在调用前，我们执行了`conn.setStreamMode(true);`，这个是为了将模式调整为流式，使得后续读数据流程更加清晰。

首先execQuery会记录各种调用信息进行相关统计，然后会进入关键的initForRequest流程，正如之前所介绍的，XSession的初始化流程是lazy的，仅分配了一个session id，然后设置状态为Init，这里就是真正创建session的流程，会发送一条SESS_NEW给server，将新session和分配的session id绑定，如果拿到的session是复用之前的，则没有这个流程（状态会是Ready）。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645616063211-3ef39859-cb35-4fa3-b9db-1cd808a8cc71.png)

然后是lazy的字符集更改，因为session可能会被回收再利用，可能会在其他请求执行中切换为其他字符集，这里会根据目标字符集和当前字符集对比，决定是否发送额外的`set names`重设字符集。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645616271654-92dd1987-84fc-487e-bd53-fb52b996a693.png)

经过一些列的变量设置，lazy DB设置，我们会构造一个用于发送具体请求的protobuf包。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645616471682-50a87e80-d493-4941-aff2-ec01a2b07793.png)

在发送的时候，有个额外的处理逻辑，这个逻辑是针对请求流水线场景下可忽略返回值的前置请求的（例如，在一个正式请求前，需要打开事务，但这条begin语句我们并不需要等待其返回，只要保证其在正式请求之前执行且不报错即可，这里我们使用expect栈功能包装前置请求和正式请求，并以流水线形式一起发出去，避免不必要的等待），这里我们没有这种前置请求，包会直接写到发送缓冲中。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645616525003-43c7631f-fccf-45e1-8cef-b1664403c581.png)

请求发送后，会同步生成一个[XResult](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java)负责结果解析，同时XResult会按照请求顺序依次拉链表，保证结果和请求一一对应。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645617013067-fe67bacf-0da1-4e5f-b07c-e0e5539af421.png)

整体请求流水线的结构如下图所示，只有处理完成前序的请求后，才能解析后续的结果。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645617150773-41bb1774-3d27-4d82-907c-20587d556e91.png)

## 接收结果集

至此我们的请求已经发送到存储节点上执行了，同时我们拿到一个[XResult](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java)，我们就是通过这个XResult来收集查询到的结果集的。

正如前文所述，XResult是和发送的请求一一对应的，同时存储节点的处理也是在会话上排队进行的，这样只需要在每个XResult中处理好自己对应的请求，就不会影响到流水线上其他请求的返回，保证流水线的正常工作。

首先我们来看下结果集处理的状态机，主要状态由获取元数据、获取数据行、获取额外信息等组成，他们之间有着比较固定的顺序，同时根据请求类型的不同，部分环节可能会被省去。报错处理是贯穿整个状态机的，任何报错信息都会导致状态机进入错误处理环节。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645670073372-72fe185c-79a2-4b7a-9ab9-1b35317f7a03.png)

对于非流式数据读取，在请求的最后会主动调用[finishBlockMode](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java#L154)将结果全部读出并缓存到[rows](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java#L113)里面，而对应上述测试代码中流式执行的情况，结果集状态机消费数据包队列则是由XResult的[next](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java#L1006)函数推动的。而具体推动状态机执行的内部函数是[internalFetchOneObject](https://github.com/ApsaraDB/galaxyglue/blob/main/src/main/java/com/alibaba/polardbx/rpc/result/XResult.java#L554)，该函数会递归调用前序的XResult，消费完前序的请求返回结果，再从数据包队列中消费并推动状态机流转。

对于`select 1`这种查询，首先会收到RESULTSET_COLUMN_META_DATA包，表示返回数据列的定义，一个包表示一列。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645671228089-f97ad481-2243-4b24-98d6-00c0ef1fd93d.png) 
元数据包之后，就会收到包含数据行的RESULTSET_ROW包了，一个包对应一行。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645671359501-c980e8d1-0ea9-42fc-8399-53041b9642ed.png) 
当全部数据行传输完成后，server端会发生一个RESULTSET_FETCH_DONE包标示结果集数据发送完成。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645671477893-2aa792d3-e23c-459a-93c1-13a775510e2f.png) 
在请求结束前，还有有个NOTICE包，用于告诉客户端rows affected或者其他信息（包括waring、generated id等）。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645671543791-5011222d-9b73-44e4-a167-61acb9bb5952.png) 
最后会有一个SQL_STMT_EXECUTE_OK包，标示着这个请求完结。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/208438/1645671710772-91d2158b-daad-4c5d-8ac7-5b8b59ac350d.png)

至此，一个完整的请求就已经处理完成了，控制台上应该也打出了`select 1`的请求结果。

# 总结

虽然本文篇幅较长，但仅仅只描述了单个简单请求的处理流程，在实际[galaxysql](https://github.com/ApsaraDB/galaxysql)的使用中还涉及多请求流水线、流控、执行计划传输、chunk结果集传输等更多高级特性的使用，相信大家通过本文的描述，基本掌握了私有协议连接流程中的关键点和关键数据结构，在调试和修改使用中也能更加得心应手。



Reference:

