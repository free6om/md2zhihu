最近排查一个问题的时候发现了MySQL JDBC驱动的setBlob有个坑，在GBK编码下工作不正常，继续深入的时候感觉这个问题还比较有意思，在这里分享给大家。

Blob在数据库里很常用，里面可以是任何的二进制内容，例如一个图片、视频之类的。

**例子一：**

我们有以下的一段程序，通过MySQL官方JDBC写入一个Blob，猜一下会发生什么：

```Java
Connection conn = DriverManager
            .getConnection(
                "jdbc:mysql://xxxxx:xxxx/d1?useUnicode=true&characterEncoding=gbk&allowMultiQueries=true",
                "xxxx",
                "xxxxx");

PreparedStatement ps = conn.prepareStatement("INSERT INTO t1 (png) VALUES (?)");
Blob blob = new Blob(new byte[] {(byte) 0xde, 0x27, 0x29, 0x3B, 0x64, 0x72, 0x6F, 0x70, 0x20, 0x74, 0x61, 0x62, 0x6C, 0x65, 0x20, 0x74,0x31, 0x3B, 0x23, 0x27});
ps.setBlob(1, blob);
ps.executeUpdate();
```

执行完我们会惊喜的发现，t1表消失了。

**例子二：**

我们使用如下代码向MySQL内写入一个Blob：
//注意，这里url里指定的是gbk编码

```Java
Connection conn = DriverManager
    .getConnection("jdbc:mysql://xxxxx:xxxx/d1?useUnicode=true&characterEncoding=gbk&allowMultiQueries=true");

PreparedStatement ps = conn.prepareStatement("INSERT INTO t1 (png) VALUES (?)");
Blob blob = new Blob(new byte[] {(byte) 0xde, (byte) 0x27, (byte) 0xc4, (byte) 0x73});
ps.setBlob(1, blob);
ps.executeUpdate();
```

但是，MySQL报了一个语法错误的异常：

```Java
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '膕')' at line 1
```

如果你是数据库使用者

记住一点就好，连接上的字符集使用utf8/utf8mb4，不要使用gbk等其他编码。

例如对于Java应用，jdbc url中需要设置：

```Java
characterEncoding=utf8
```

使用gbk等编码会有几个风险：

1.  数据写不进去
1.  写进去的和查出来的不一样
1.  SQL注入的漏洞（比如注入一个DROP DATABASE）

## 如果你是数据库内核开发者

作为一个内核开发者，我们得搞明白这到底是为什么。

我们先记住几个比较特殊的字符的ASCII码：

<table>
<tr class="header">
<th></th>
<th></th>
<th>十六进制</th>
<th>十进制</th>
</tr>
<tr class="odd">
<td>单引号</td>
<td>’</td>
<td>0x27</td>
<td>39</td>
</tr>
<tr class="even">
<td>斜杠</td>
<td>\</td>
<td>0x5c</td>
<td>92</td>
</tr>
</table>

### MySQL JDBC中setBlob的实现

我们要搞明白，客户端发送给MySQL的究竟什么东西。

MySQL JDBC的setBlob，会在Blob的两头套上单引号，同时将Blob中出现的单引号(0x27)与斜杠(0x5c)使用斜杠(0x5c)进行转义。

上面的例子中，我们传入的Blob的内容是：

```Text
0xde 27 c4 73
```

那么经过驱动处理之后，传输到MySQL的字节流是：

```Text
0x49 4e 53 45 52 54 20 49 4e 54 4f 20 74 31 20 28   70 6e 67 29 20 56 41 4c 55 45 53 20 28
  I  N  S  E  R  T     I  N   T  O     t  1     (   p  n  g  )     V  A  L  U  E  S     (

0x27 de 5c 27 c4 73 27                                      
  '                 '  

0x29
  )
```

可以看到，原始数据中的0x27前被转义成了0x5c 0x27。

### GBK编码

对于GBK，我们知道以下几点就可以了：

1.  有的字符的是一个字节的，有的是两个字节的。根据第一个byte可以判断出是否有后续的byte。
1.  对于单引号与斜杠，其GBK编码与其ASCII编码一致。
1.  有很多双字节GBK字符的第二个字节是0x5c（斜杠），可以用这种方法找到很多例子：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1639368790767-b839360c-8fca-4199-91ab-d693b9eeb083.png)

### MySQL的Lexer如何对字节流进行解析

我们再来看MySQL收到上文中的字节流后，是如何进行解析的。

我们会发现在对引号内的数据进行解析之前，MySQL是无法“提前”判断出以下几个信息的：

1.  内容的长度
1.  内容是一个符合连接编码的合法字符串，还是一个可以包含任意数据的Blob

按照MySQL目前的做法，会逐个byte来对收到的字节流进行解析，对于每个byte：

1.  会先尝试按照连接上的编码进行解析
1.  如果不是合法的字符，则直接保存该byte

这里有个需要注意的地方在于，有很多字符集都是多字节的（例如GBK、UTF8MB4等），Lexer会根据当前的byte的值，来判断是否有后续的byte。以GBK编码为例，MySQL使用如下的条件判断当前byte是一个字符，还是和后面的一个byte共同组成一个字符：

```C++
#define isgbkhead(c) (0x81<=(uchar)(c) && (uchar)(c)<=0xfe)
```

我们使用上面的例子模拟下这个过程，MySQL看到的是“de 5c 27 c4 73”：

1.  首先处理第一个byte，0xde，由于(0x81<=0xde<=0xfe)，Lexer会认为0xde与后面的0x5c共同组成了一个字符，并且，0xde 0x5c确实是一个合法的GBK字符“轡”，于是Lexer将“轡”储存起来，并跳过了前两个byte。
1.  处理第三个byte，0x27，这个字符是单引号，和前面的单引号完成了配对，该字符串的解析就结束了

这时候你会发现，实际上后面的内容还没有被处理，却被当成了字符串之外的内容，这些内容显然是非法的，于是Parser就报了语法错误。

这里的原因在于，本来充当转义符的0x5c，被0xde给吞掉了。

### 随机测试结果

我们分别使用UTF8与GBK编码连接数据库，同时随机生成Blob，调用setBlob方法进行写入，结果是：

1.  GBK编码下，写入很容易失败，报错五花八门，不外乎语法错误之类的。同时，即使写入成功的一些Blob，查出来和写进去的值很多存在差异（读者可以自己想下为什么）。
1.  UTF8编码下，写入都是成功的，并且查出来的值和写进去的值也是一模一样的。

GBK下有问题我们可以理解了，但为什么UTF8下没有问题呢？

### UTF8恰好能用

setBlob在UTF8编码下工作的很正常，是一种典型的“程序恰好能用”的结果，如下：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1639368867287-69d82118-cf25-4b5f-ae57-777bc028051b.png)

我们先看下utf8/utf8mb4的几个于此相关的特点：

1.  不同的字符长度不同，最多4个字节，最少1个字节。根据第一个byte可以判断出是否有后续的byte。
1.  单字节字符与ASCII码一致
1.  多字节的UTF8字符，其每个字节的区间不会包含0x27与0x5c

举个例子，“孙”在UTF8编码里是一个三字节的字符：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1639368879644-4f3ccb02-00ab-4def-8530-fc8f8b3c49f6.png)

我们构造这样一个Blob：

```Text
0xe5 ad 27 c4 73
```

按照上文描述，setBlob会做如下转义：

```Text
0xe5 ad 5c 27 c4 73
```

MySQL的解析过程：

1.  MySQL的Lexer在处理0xe5 与 0xad的时候，会识别出这是一个三字节的字符，但它在看第三个字节5c的时候，会发现0xe5ad5c不是一个合法的utf8字符，示意：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1639368914878-97e79f69-44bc-4dba-9cce-3fa72277f6b5.png)

于是MySQL会将前两个byte保存下来，并不会吞掉第三个byte。
2. 处理第三个byte 0x5c，此时他能正确的起到转义符的作用
3. 第四个byte 0x27，被正确的转义
4. 后续的byte也能正常的当做字符串的一部分被处理

于是这个Blob能够正常的被写入。

所以最后发现，为什么setBlob在utf8编码下工作正常，就是因为“多字节的UTF8字符，其每个字节的区间不会包含0x27与0x5c”， 也就不会导致转义符0x5c或者引号0x27被“吃掉”的情况。

### 更严重的后果

与单纯的写入失败相比，转义符被吞实际上很容易构造出一些SQL注入例子，这种注入可以完全绕过PrepareStatement的保护和目前已有的注入检测工具（例如Druid的SQL Wall）。

正如我们第一个删表的例子。

### 谁的锅

这个问题根源上说是MySQL设计的不太严谨，存在一些自相矛盾的地方（普通字符串和binary字符串的转义方式不同，但语法确实一样的）导致的。

不过更大的责任应该是MySQL JDBC驱动setBlob实现的有BUG。

事实上，驱动中有一个与setBlob方法功能类似的方法叫setBytes，这个方法的实现则非常的安全可靠，与编码无关。他是将Blob改成了十六进制字符串的形式拼到了SQL里，例如上述的例子：

```SQL
INSERT INTO t1 (png) VALUES (0xde27c473)
```

这样的解析对Lexer就非常友好了。

但是，很多框架，例如Hibernate，内部已经写死了使用setBlob方法，这样就必须记住连接使用utf8编码了。

此外，其他语言的驱动我们没有做太多研究，不清楚是否存在类似的问题。

## PolarDB-X对Blob的支持

PolarDB-X虽然是一个自研的分布式数据库，但对字符集（包括Collation）做了很全面的支持。例如常见的utf8\utf8mb4\gbk\latin1编码等。比一些常见的同类产品支持的更为全面。

由于setBlob很常用（并且很多时候是不得不用），我们的Lexer在utf8/utf8mb4编码下兼容了MySQL的行为，确保Blob能正常的写入。

详见：https://help.aliyun.com/document_detail/316560.html

## 题外话

从MySQL的变更记录来看，实际上早年的MySQL（4.1.0以前）的Lexer实现的比现在要简单的多，它没有去按字符集解析字符串常量，而是按照byte(或者说ascii)去解析，也即所有的字符都按照1个byte的长度去处理。

这样实现的好处看，实现起来很简单，支持utf8编码没啥问题。但是如果要支持gbk等编码（因为gbk字符可能编码出转义符斜杠），就有些困难了。

这里有两种路径：

1.  在客户端对gbk字符编码出来的特殊ascii码（例如0x5c）进行转义
1.  在服务端按照编码进行解析，而不是按ascii进行解析

如上文，MySQL选择的是第二种路径。个人理解这里的原因是第二种的性能更好，少了一遍转义的流程。

从MySQL驱动的代码也可以看到这个发展过程，对于<4.1.0的MySQL，由字符串编码而生成的0x5c会被转义，而对于高版本的MySQL，则不需要被转义。

像开源数据库TiDB，其Lexer也是基于ascii去做的解析，这可能是TiDB目前只能支持utf8/latin1的一个原因。如果TiDB要支持gbk等编码，目前也只能走第二种路径（因为走的是使用mysql驱动的路线，驱动的行为已经定死了）。

## 备注

本文测试的MySQL是5.7的版本，驱动使用的是5.1.40。MySQL 8.0与最新的8.0.x的驱动代码也看过，此问题看起来并没有被修复。



Reference:

