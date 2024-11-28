
# 1 Redis 的简介


Redis 实际上是简称，全称为 `Remote Dictionary Server` (远程字典服务器)，由 Salvatore Sanfilippo 写的高性能 key\-value 存储系统，其完全开源免费，遵守 BSD 协议。Redis 与其他 key\-value 缓存产品（如 memcache）有以下几个特点。


* 数据持久化：可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。


* 数据结构简单丰富：既有简单的 key\-value 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。


* 高可用：支持主从、哨兵、集群等模式，可以有效提高可用性。


Redis 也是一种 分布式缓存 ，其代码是 c 语言写的，那我们该如何阅读呢？


# 2 环境搭建


环境依赖，先看看 gcc 、cc、g\+\+ 有没有安装



```
whereis gcc
whereis cc
whereis g++

```

安装gcc



```
xcode-select --install  
brew install gcc  
brew install pkg-config

```

查看 gcc 的版本：



```
$ gcc --version
Apple clang version 14.0.0 (clang-1400.0.29.202)
Target: x86_64-apple-darwin22.1.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin

```

我使用 `CLion 2022.3.1` ，这个版本可以支持 `Makefile` 的项目，我们可以检查一下环境是不是有问题, 如果有问题，这里会有错误信息，**我的之前报错是因为 Clion 的版本版本太低了，升级之后就好了。**


[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002858.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002858.png)


下载Redis源码：



```
git clone https://github.com/redis/redis.git

```

切换到指定的版本



```
git checkout 7.0

```

`File => New CMake Project from Sources`, 打开源码项目, 会自动生成根目录下的 `CMakeList.txt` 文件:
Clion 导入项目的时候选择已有的 MakeFile 文件，如果有是否 `clean` 项目，选择 `clean` 即可，之后可以点开 `MakeFile` 文件：


[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105037.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105037.png)


如果需要禁止编译器优化，可以使用下面命令：



```
make CFLAGS="-g -O0" MALLOC=jemalloc

```

运行完之后， Src 文件下就会出现可运行文件：
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105126.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105126.png)


然后可以看到这些可运行的选项，继而配置`Edit configuration` 运行配置：


[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105144.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230921105144.png)


[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002538.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002538.png)


选择 `debug` 进行启动，启动成功，然后可以进行调试了：
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002603.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002603.png)


可以使用 `Redis Desktop Manager` 来进行连接：
[![image.png](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230701155636.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230701155636.png)


或者命令行连接（**没有密码就可以不需要 \-a 12345**）：



```
redis-cli -h 127.0.0.1 -p 6379 -a 12345

```

如果头文件引入报红色下划线，那就试试重新加载一下
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002630.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002630.png)


# 3 Redis源码阅读技巧


## 3\.1 Redis 的目录结构


Redis 的目录：


* deps: Redis 所依赖的第三方代码库
	+ hdr\_histogram：用于生成命令的延迟追踪直方图
	+ hiredis：官方c语言客户端
	+ Jemalloc：内存分配器，默认情况下选择该内存分配器来代替 Linux 系统的 libc\-malloc，libc\-malloc 性能不高，且碎片化严重。
	+ linenoise：一种读线替换。它由 Redis 的 同一作者开发，但作为一个单独的项目进行管理，并根据需要进行更新。
	+ lua：lua 脚本相关的功能。
* src：源代码
	+ commons：都是 json 文件，放着每个指令的原信息。
	+ modules：实现 Redis Module 的示例代码。
	+ 其他文件均是源码
* test：测试代码
	+ cluster，Redis Cluster 功能测试。
	+ sentinel，哨兵集群功能测试。
	+ unit，单元测试。
	+ integration，主从复制功能测试。
* utils：工具类
* Makefile：编译文件
* redis.conf : redis 启动的配置文件
* sentinel.conf：哨兵配置


## 3\.2 Redis 源码阅读顺序


网上的源码阅读顺序（引自网上）：


* 自底向上：从耦合关系最小的模块开始读，然后逐渐过度到关系紧密的模块。就好像写程序的测试一样，先从单元测试开始，然后才到功能测试。
* 从功能入手：通过文件名（模块名）和函数名，快速定位到一个功能的具体实现，然后追踪整个实现的运作流程，从而了解该功能的实现方式。
* 自顶向下：从程序的 main() 函数，或者某个特别大的调用者函数为入口，以深度优先或者广度优先的方式阅读它的源码。


从大方向来说，学习 Redis 会有两种路径：


* 先从数据机构入手，直接手撕数据结构
	+ 好处：学着踏实，知根知底
	+ 坏处：容易从入门到放弃
* 先从启动 Redis 开始，跟着启动顺序读源码，跟着具体的操作读源码
	+ 好处：比较符合人的认知路线，知道 Redis 启动做了哪些操作，执行命令时做了哪些操作。
	+ 坏处：容易迷路，前期看哪一句，都不知道在干嘛，毕竟 RDB，AOF，集群，哨兵这些源码，如果实操过才相对容易理解一点。


个人建议是先学习如何启动 Redis，抓大放小（大致知道哪个类启动，读那些配置文件，大概是做什么用的），学习 Redis 到底能干什么，大致知道 Redis 的一些用法之后，再去了解 Redis 的常用的数据结构，到底怎么实现的，这个时候对 Redis 的一些数据结构大致有印象，之后可以跟着 Redis 启动，执行命令去看具体功能执行的路径。
在 Debug 的过程中，可以加深影响，更加了解数据结构的设计，代码的调用关系。


# 4 C语言的知识


## 4\.1 \#define的基本用法


在C语言中，常量是使用频率很高的一个量。常量是指在程序运行过程中，其值不能被改变的量。常量常使用 `#define`来定义。
使用`#define`定义的常量也称为符号常量，可以提高程序的运行效率，Redis 的源代码中有比较多的地方都使用该方式。
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002700.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002700.png)


一般有以下两种用法：



```
#define 宏名 宏值
#define 宏名(参数列表) 表达式

```

第一种就是定义常量，比如：



```
#define N 100

```

此后直到 `#undef N`之前， N的值都是100。当遇到`#undef N`，其后如果再出现 N，则 N 需要重新定义之后才可以使用。


第二种语法常用来定义符号函数。
例如：



```
#define AREA(x,y) (x)*(y)

```

表示用来求长和宽分别是x和y的矩形的面积。
需要注意的是，在表达式(x) \* (y)中，x和y都要使用“（）”括起来，这是因为符号函数在编译时时进行符号形式替换。如果不加（）则可能会发生意想不到的错误，例如：



```
#define AREA(x,y)  x*y
...
A = AREA( 2+3, 1+2 );

```

此处预期的结果是15，但是实际的结果却是7，这是因为该段代码在编译进行了简单的符号替换而得到的实际表达式是：
`A = 2+3 * 1+2;`


根据运算符的优先级，先进行乘法运算，然后才是加法，这就导致了错误。
而如果使用



```
#define AREA(x,y)  (x)*(y)
...
A = AREA( 2+3, 1+2 );

```

则在编译时替换的结果是：



```
A = (2+3) * (1+2);

```


```
#include"stdio.h"  
#define AREA(x,y)  (x)*(y)  
int main()  
{  
    int a = AREA(2+3, 1+2);  
   printf( " %d\n", a);  
   return 0;  
}

```

## 4\.2 头文件


Redis 是使用 c 语言写的，里面有很多头文件:



```
#include "server.h"  
#include "monotonic.h"  
#include "cluster.h"  
#include "slowlog.h"  
#include "bio.h"  
#include "latency.h"  
#include "atomicvar.h"  
#include "mt19937-64.h"  
#include "functions.h"  
#include "syscheck.h"  
  
#include 

```

以 `<` 开头的，比如 `#include`  是标准库的头文件，会在系统指定路径下查找，对应到 `Java`里面可以理解为 官方的 jdk 里面的类，而类似 `#include "server.h"`  则是工程里面自定义的。


我没怎么写过 c 语言的代码， 一般 `.c` 文件是写实现的代码逻辑的，那如何在 a 文件里面写一个方法，让 b 文件也能用呢？


通过头文件的机制，类似 Java 里面的 接口， `public` 和 `private` 的概念，Java 中 一般希望对外暴露的方法，会设置为 `public` ，，如果不希望暴露，则设置为`private`。c 语言里面如果希望暴露，则可以在头文件里面定义，否则不用定义。（虽然c语言是面向过程的，但是Redis确实在里面实践一些面向对象的思想）。


比如计算两数之和 与 两数之差 的乘积 `test.c`



```
long long mul(int a,int b) {  
    return a*b;  
}  
  
  
long long calculate(int a,int b) {  
    return mul(a+b,a-b);  
}

```

暴露出去的头文件`test.h`



```
long long calculate(int a,int b);

```

运行的代码 `main.c` ，可以正常计算结果为 `-3`:



```
#include "stdio.h"  
#include "test.h"  
int main(){  
    printf("结果：%lld",calculate(1,2));  
    return 0;  
}

```

但是如果直接引用 `sum()` 方法，则会报错，无法使用：
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002719.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002719.png):[slower加速器官网](https://chundaotian.com)


如果我们多次引用头文件会怎么样？结果是正常运行：
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002734.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002734.png)


## 4\.3 ifndef


Redis 里面有挺多的地方定义头文件的时候总是来一句 `#isdef` 或者 `ifndef`



```
#ifdef __linux__  
#include   
#endif

```


```
#ifndef __ADLIST_H__  
#define __ADLIST_H__
...
#endif /* __ADLIST_H__ */

```

如果加了 `#ifndef` ，则会判断只有没有定义这个宏的时候，才会定义它，第二次再次遇到 `include` 的时候，发现这个宏已经被定义过了，就会直接跳过，这样可以保证多次 `include` 也不会被解析多次，有且只有一次。


解析多次的坏处是什么？


1. 如果在`.h` 文件里面定义了全局变量，会导致变量重复定义。这个基本不太会，公司编码规范一般都会禁止，这样写是不人道的。
2. 浪费编译时间。


既然禁止了在 `.h` 文件里面定义全局变量，那全局变量在哪里定义呢？当然是 `.c` 文件，比如 Redis 里面的全局变量：
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002750.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002750.png)


那其他的文件怎么使用？这个 `sever` 可是全局唯一的，维护了 `redis` 的全部状态数据，那当然是暴露出去，在哪里暴露出去，在 `.h` 文件，使用关键字 `extern`
[![](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002805.png)](https://markdownpicture.oss-cn-qingdao.aliyuncs.com/blog/20230922002805.png)


# 5 小结一下


阅读源码，是一件长期的事情，但是我们每次跟读代码的时候，一定要带着问题去阅读，否则效率会下降挺多。前期了解数据结构模型的时候，可以在网上找一些简单易懂的博客，最好是有图片的，书籍比较推荐《Redis 设计与实现》。有一定了解之后，会有些疑问，不用担心，此时再通过读源代码去验证我们的想法，可能不少小伙伴没学过 c 语言，也不必担心，语言之间都是相通的，其次即使有关键字不会，可以通过搜索也可以快速了解其作用。
希望我们都能从全局看功能 \-\-\> 实践 \-\-\> 抓大放小 \-\-\> 带疑问看源码 \-\-\> 重构知识图谱 \-\-\> 关联知识 \-\-\> 跳出细节俯瞰全局，最终完成 Redis 相关的知识学习，并形成一套自己的方法论。


作者：[秦怀](https://github.com)


\_\_EOF\_\_

![](https://github.com/Damaer/p/18573298.html)秦怀杂货店 本文链接：[https://github.com/Damaer/p/18573298\.html](https://github.com)关于博主：评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。版权声明：本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！声援博主：如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。您的鼓励是博主的最大动力！
