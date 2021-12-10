本文主要讲解PolarDB-X的CN节点（GalaxySQL）的启动过程包括参数加载、元信息加载等过程并对启动过程中设计的模块做简单的介绍。

CN Server层的代码主要包含在polardbx-server模块中，main函数位于`TddlLauncher`。

主逻辑入口在`CobarServer.init()`方法中。

CN启动分为以下几个流程：

## CobarServer对象的创建

该类是单例，可以通过`CobarServer.getInstance(`获取。

## 参数加载

路径：`TddlLauncher.main()` -> `CobarServer.new()` -> `CobarConfig.new()` -> `CobarConfig.initCobarConfig()` -> `ServerLoader.load()`

CN的参数均为key-value的形式。

在`ServerLoader.load()`中，CN主要从以下位置读取参数：

-   server.properties文件，其默认位置在classpath的根目录中，所以一般使用IDE进行开发时，加载的是polardbx-server\src\main\resources\server.properties文件

```
String conf = System.getProperty("server.conf", "classpath:server.properties");
```

-   Java运行参数中

例如 java ... -DserverPort=8527

-   环境变量中

```
serverProps.putAll(System.getProperties());
serverProps.putAll(System.getenv());
```

以上参数来源中，server.properties的优先级最低，环境变量中的优先级最高，当包含同名参数时，高优先级的参数来源会覆盖低优先级的参数来源。

加载的参数，会保存在`SystemConfig`中，该类为单例，可以通过`CobarServer.getInstance().getConfig().getSystem()`获取。

## 从MetaDB读取元数据，并初始化实例级的系统组件

路径：`TddlLauncher.main()`->`CobarServer.new()`->`CobarConfig.new()`->`CobarConfig.initCobarConfig()`->`ServerLoader.load()`->`ServerLoader.initPolarDbXComponents()`

### 初始化元数据库的连接池

`MetaDbDataSource.initMetaDbDataSource`

会使用`SystemConfig`中存储的MetaDB的地址、端口、用户名、密码、库名等信息，建立与MetaDB的连接。

```
 MetaDbDataSource
            .initMetaDbDataSource(this.system.getMetaDbAddr(), this.system.getMetaDbName(),
                this.system.getMetaDbProp(),
                this.system.getMetaDbUser(),
                this.system.getMetaDbPasswd());
```

`MetaDbDataSource`是一个单例，实现了JDBC的程序中可以使用`MetaDbDataSource.getInstance().getConnection()`获取与MetaDB的连接（实现了JDBC接口），并使用该连对MetaDB进行访问。

### 对系统表进行创建或者升级

```
 SchemaChangeManager.getInstance().handle();
```

polardbx-gms\src\main\resources\ddl\中保存了系统表的表结构，并且使用alter语句记录了每一次版本的变更。`SchemaChangeManager`在初始化时，会检测当前每个系统表的表结构版本，如果版本较老，会依次使用这些语句对alter语句进行更新。

### 读取实例ID信息

```
ServerInstIdManager.getInstance();
```

一个PolarDB-X集群也称为一个实例，在MetaDB中有一个唯一的实例ID。同时，PolarDB-X实例有主实例与只读实例两种模式。当存在只读实例时，必然存在一个主实例。该步骤从MetaDB中读取实例ID，如果当前实例为只读实例，则还会读取其主实例的实例ID。

### MetaDbConfigManager

先简单介绍下这个东西是干啥的。

我们存在MetaDB中的配置信息，如果发生变化，CN需要能感知到这个变化。例如，某个表增加了一个列。作为CN，如何能感知到这个变化呢？

一个比较简单的思路是对元数据做轮询，各几秒查一次。但如果让每个模块都去做这样的事情，会有个比较大的问题是对MetaDB的访问压力会很大。

`MetaDbConfigManager`对此作了封装，作了一个统一的轮询机制。但由于每个模块元数据表的格式千变万化，所以`MetaDbConfigManager`并不是直接轮询每个模块的元数据表，而是轮询config_listener这张表。这张表的几个关键列是data_id、op_version、gmt_modified。

我们可以为每个data_id在代码中注册一个listener，当`MetaDbConfigManager`轮询到data_id的op_version发生变化的时，会回调这个listener，一般情况下，各模块实现的listener会按需要再读取对应的元数据表。

例如，d1.t1表在config_listener中有一行记录：

```
 polardbx.meta.table.d1.t1
```

往t1表中增加了一个列，我们会修改MetaDB中的columns表，同时，我们会修改config_listener表中 polardbx.meta.table.d1.t1这行记录的op_version，进行+1的操作。

几秒钟后，`MetaDbConfigManager`会轮询到这行记录的op_version列发生了变化，会回调表结构管理模块的listener（类：`TableMetaListener`），该listener会重新加载d1.t1表的元数据。

所以`MetaDbConfigManager`可以做到每个CN节点只有一个线程对MetaDB做轮询就能感知到各个元数据表的变化。

在启动时，CN会初始化`MetaDbConfigManager`用于做轮询的线程和定时任务。

### MetaDbInstConfigManager

初始化实例级的配置项，这里的配置项类似于MySQL中的System Variables，例如SQL的内存大小限制之类的。该阶段会从MetaDB的inst_config表中加载所有的配置项并保存在内存中。

并且会注册对应的listener，这样当inst_config表发生变化的时候，会回调`MetaDbInstConfigManager`的listener。

### ConnPoolConfigManager

这个是用来存连接池（CN与DN之间的连接池）相关的配置项的（其实它用到的配置`MetaDbInstConfigManager`中都有），也是从inst_config中读取。

### StorageHaManager

这个比较重要。

目前每一组DN（这里的组指一个Paxos组）内选举的结果是存储在该组DN自己的系统表中，并不会写到MetaDB中。因此CN需要有机制去从DN的系统表中探测各节点的角色。

`StorageHaManager`会从storage_info表中读取所有的DN节点的连接信息，并且内部会有轮询的线程去检测角色信息。
当DN发生HA的时候，`StorageHaManager`会探测到这个变化，并感知到最新的Leader等角色

### 初始化系统库

`initSystemDbIfNeed`初始化information_schema和polardbx等几个系统库。

## 创建线程池

路径：`CobarServer.new()`

几个主要的线程池的创建：

-   managerExecutor，负责Manager端口的请求
-   killExecutor，专门用来执行kill指令
-   serverExecutor，各种SQL都是由这个线程池执行

## CobarServer.init

路径：`TddlLauncher.main()`->`CobarServer.init()`

### 逻辑库（TDataSource）的初始化

入口在`GmsAppLoader.initDbUserPrivsInfo`，类里的“App”指的就是一个逻辑库。

这里会加载每个逻辑库的用户权限信息以及最重要的TDataSouce。

`TDataSource`在CN中与逻辑库一一对应。它是每个逻辑库执行SQL的入口。

`TDatSource`的初始化逻辑在`MatrixConfigHolder.init`中，包含了以下内容：

-   初始化拓扑，例如这个库包含了哪些DN，并初始化与这些DN之间的连接池（`TopologyHandler.initPolarDbXTopology`）
-   获取每个DN的信息，包括版本，特性的支持程度等（`StorageInfoManager.init`）
-   初始化分片的路由信息（mode='drds'，`TddlRuleManager.init`；mode='auto'，`PartitionInfoManager.init`）
-   初始化表管理器（有哪些表、每个表有哪些列哪些索引等）（`GmsTableMetaManager.init`）
-   初始化事务管理器（`transactionManager.init()`）
-   创建Plan Cache（`PlanCache planCache = new PlanCache(schemaName);`）
-   启动DDL任务引擎（`ddlEngineInit()`）
-   统计信息管理器的初始化（`StatisticManager.init`）
-   SPM的初始化（`PlanManager.init`）

### GmsClusterLoader.loadPolarDbXCluster

加载集群信息，例如集群中有哪些CN节点等。

初始化`CclService`，这个是用来做SQL限流的。

### warmup

CN的SQL函数目前是反射机制进行加载的，这里主要作用是加载下所有的函数。

### 网络层的初始化

该步骤结束后，服务端口便会打开，该CN进程就能开始对外提供服务了。

CN在这一步会启动Server端口与Manager端口两个端口。

Server端口对应MySQL的3306端口，是提供给前端应用使用的。

Manager端口用于内部的管理使用，例如，SHOW PROCEESSLIST指令需要收集所有CN的执行情况，就会使用该端口进行收集。

#### NIOProcessor

```
processors = new NIOProcessor[system.getProcessors()];
for (int i = 0; i < processors.length; i++) {
      processors[i] = new NIOProcessor(i, "Processor" + i,this.serverExecutor);
processors[i].startup();
}
```

CN网络层处理请求的入口是N`IOProcessor`，每个`NIOProcessor`有读写两个线程。这里会来启动`NIOProcessor`的W与R线程。

`NIOProcessor`只负责网络相关的处理，如果接收到的是SQL请求，后续的SQL的执行则会交予ServerExecutor线程池进行执行。

#### NIOAcceptor

CN网络层用来处理连接建立请求的是`NIOAcceptor`，`NIOAcceptor`的new过程中，会打开端口进行监听：

```
public NIOAcceptor(String name, int port, FrontendConnectionFactory factory, boolean online) throws IOException {
        super.setName(name);
    	...
            this.selector = Selector.open();
            this.serverChannel = ServerSocketChannel.open();
            this.serverChannel.socket().bind(new InetSocketAddress(port), 65535);
        ...
    }
```

同时，`NIOAcceptor`也是一个线程，会处理连接建立的请求。当连接建立后，`NIOAcceptor.accept`方法会将连接绑定到一个`NIOProcessor上`，由`NIOProcessor`继续处理该连接上后续的读写请求。

可以看出，对于服务端口来说，`NIOAcceptor`只有一个，`NIOProcessor`的数目则一般和CPU核数保持一致。

### MPP Server的启动

CN作为MPP集群中的一个，还需要启动端口来进行CN之间的通信。

`startMppServer`会进行MPP服务的启动。

### CDC服务的启动

`CobarServer.tryStartCdcManager`会进行CDC的启动。

## 结语

至此，CN就启动完成了。这个过程包含了CN中绝大多数组件，鉴于篇幅原因，没有完整介绍每个组件的作用。如果你对这些组件中的哪一个感兴趣，欢迎留言给我们，我们会在后续的文章中，对一些关键的组件做进一步详细的介绍。



Reference:

