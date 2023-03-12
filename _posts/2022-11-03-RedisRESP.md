---
layout: post
title: 【Redis】RESP通信协议
---

RESP是**REdis Serialization Protocol**的简称，即 redis 序列化协议；也就是专门为redis设计的一套序列化协议。这个协议其实在redis的1.2版本时就已经出现了,但是到了redis2.0才最终成为redis通讯协议的标准。

## 网络通信

我们知道，在传统计算机网络模型中，传输层（TCP / UDP）的上一层便是应用层。应用层协议一般专注于数据的编解码等约定，比如经典的 HTTP 协议。

RESP 协议本质和 HTTP 是一个级别，都属于应用层协议。

在 redis 中，传输层协议使用的是 **TCP**，服务端从 TCP socket 缓冲区中读取数据，然后经过 **RESP 协议解码**得到我们的指令。

而写入数据则是相反，服务器先将响应数据使用 RESP 编码，然后将编码后的数据写入 TCP Socket 缓冲区发送给客户端。


## 协议格式

在 RESP 协议中，第一个字节决定了具体数据类型：

1. `简单字符串`：Simple Strings，第一个字节响应 `+`
2. `错误`：Errors，第一个字节响应 `-`
3. `整型`：Integers，第一个字节响应 `:`
4. `批量字符串`：Bulk Strings，第一个字节响应 `$`
5. `数组`：Arrays，第一个字节响应 `*`



**exp1：**

命令”set key1 1“ 一般被序列化成：

```
*3\r\n$3\r\nset\r\n$4\r\nkey1\r\n$1\r\n1\r\n
```

一般人看不懂，为了方便理解，格式化上面的例子

```
*3\r\n    --这个命令包含3个（bulk）字符串
$3\r\n    --第一个bulk string有3个字节
set\r\n   --第一个bulk string是set
$4\r\n    --第二个bulk string有4个字节
key1\r\n  --第二个bulk string是key1
$1\r\n    --第三个bulk string有1个字节
1\r\n     --第三个bulk string有1
```

**exp2：**

命令”get key1“：

```
*2\r\n$3\r\nget\r\n$4\r\nkey1\r\n
```

格式化上面的例子变成

```
*2\r\n    --这个命令包含2个（bulk）字符串的数组
$3\r\n    --第一个bulk string有3个字节：get
get\r\n
$4\r\n    --第二个bulk string有4个字节：key1
key1\r\n
```



## 内联命令

`Inline commands`。是这样的，一般情况下我们和 redis 服务端通信都需要一个客户端（比如`redis-cli`），因为双方都遵循 RESP 协议，数据可以正常编码和解析。

考虑这样一种情况，当你没有任何客户端工具可用时，是否也能正常和服务端通信呢？比如 **telnet**。

也是可以的，redis 正式通过 **内联指令** 支持的，咱们来看看例子：

例1，通过 RESP 协议发送指令（由于没有客户端，这里我们手动编码）：

```bash
[root@VM-20-17-centos ~]# telnet 127.0.0.1 6379
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
*3       
$3
set
$4   
key1
$5 
world
+OK

get key1
$5
hello
```

我们正常的指令是 `set key1 word`，经过 RESP 编码之后 `*3\r\n$3\r\nset\r\n$4\r\nkey1\r\r$5\r\nworld`，redis 服务端解码之后便可得到正常指令。

例2，通过内联操作发送指令：

```bash
[root@VM-20-17-centos ~]# telnet 127.0.0.1 6379
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
exists key1
:1
get key1
$1
1
set key1 hello             
+OK
get key1
$5
hello
```

这里我们直接发送 内联指令 比如 EXISTS key1、GET key1、SET key1 hello 等，无需 RESP 协议编码，服务端仍可正常处理。

值得注意的是，因为没有了统一请求协议中的 * 项来声明参数的数量，所以在 telnet 会话输入命令的时候，必须使用空格来分割各个参数，服务器在接收到数据之后，会按空格对用户的输入进行解析，并获取其中的命令参数。





## 高性能 Redis 协议解析器

**High performance parser for the Redis protocol**，即，高性能 Redis 协议分析器。

RESP 是一款**人类易读、简单实现**的通信协议，它可以类似于二进制协议的性能实现。

RESP 使用**前缀长度**来传输批量数据，因此不需要像 JSON 那样，为了查找某个特殊字符而扫描整个数据，也无须对发送至服务器的数据进行转义。

程序可以在对协议文本中的各个字符进行处理的同时， 查找 CR 字符， 并计算出批量回复或多条批量回复的长度， 就像这样：

```c
#include <stdio.h>

int main(void) {
    unsigned char *p = "$123\r\n";
    int len = 0;

    p++;
    while(*p != '\r') {
        len = (len*10)+(*p - '0');
        p++;
    }

    /* Now p points at '\r', and the len is in bulk_len. */
    printf("%d\n", len);
    return 0;
}
```

得到了批量回复或多条批量回复的长度之后， 程序只需调用一次 read 函数， 就可以将回复的正文数据全部读入到内存中， 而无须对这些数据做任何的处理。

在回复最末尾的 CR 和 LF 不作处理，丢弃它们。

Redis 协议的实现性能可以和二进制协议的实现性能相媲美，并且由于 Redis 协议的简单性，大部分高级语言都可以轻易地实现这个协议，这使得客户端软件的 bug 数量大大减少。



## 总结

协议，本质是双方对数据处理的一种约定，redis 提供了简单易实现的 RESP 协议，你也看到了，确实相当简单，按照这种协议约定，你也能很快写出一个 redis 客户端。

协议工作的一般流程是：

- 客户端：原始命令 -> RESP 编码
- 服务端：RESP 解码 -> 原始命令

redis 服务端除了支持 RESP 协议，还支持 内联指令，也就是我们原始的命令，这样一来就不需要编码解码的过程了。

**RESP3** 提供了更清晰、丰富的数据类型，感兴趣可以点击[详情](https://github.com/antirez/RESP3/blob/master/spec.md)