提纲
==========
***

1. [DistributeFS的概要设计](#DistributeFS的概要设计)
  1. [一个简单到不能再简单的概要设计](#一个简单到不能再简单的概要设计)
  1. [系统的组成模块](#系统的组成模块)
  1. [FSClient的设计](#FSClient的设计)
1. [开发环境/技术依赖](#开发环境)
1. [interfaces](#interfaces)
  1. [Block是基本的数据结构](#Block是基本的数据结构)
  1. [BlockServer与BlockMaster的通讯接口](#BlockServer与BlockMaster的通讯接口)
  1. [BlockMaster的服务接口](#BlockMaster的服务接口)
  1. [BlockServer的服务接口](#BlockServer的服务接口)
1. [Block的第一个版本的实现](#Block的第一个版本的实现)
  1. [先写单元测试](#先写单元测试)
  1. [编码是一件愉快的事情](#编码是一件愉快的事情)
  1. [对代码风格的理解](#对代码风格的理解)
  1. [Block是必须是独立完美的](#Block是必须是独立完美的)
1. [BlockServer：容器与IO](#BlockServer：容器与IO)
  1. [BlockServer:Block的容器](#BlockServer:Block的容器)
  1. [BlockServer:数据的存取服务](#BlockServer:数据的存取服务)
  1. [设计模式让程序迸发智慧的光芒](#设计模式让程序迸发智慧的光芒)
1. [实现第一个版本的FSClient](#实现第一个版本的FSClient)
  1. [呈现给用户的接口一定要简单易用](#呈现给用户的接口一定要简单易用)
  1. [越过障碍：mock一个BlockMaster](#越过障碍：mock一个BlockMaster)
  1. [第一次Server、Client联调](#第一次Server、Client联调)
1. [BlockMaster的实现](#BlockMaster的实现)
  1. [BlockServer的详细设计](#BlockServer的详细设计)
  1. [BlockServer数据结构设计](#BlockServer数据结构设计)
  1. [坚守代码质量](#坚守代码质量)
  1. [设计BlockServer备份节点](#设计BlockServer备份节点)
1. [联调](#联调)
  1. [用BlockMaster替掉mock的那个](#用BlockMaster替掉mock的那个)
  1. [第二次Server、Client联调](#第二次Server、Client联调)
  1. [分布式系统初步可用](#分布式系统初步可用)
1. [补课](#补课)
  1. [config是一个重要的模块，而不是一个static final域]
  1. [Block的压缩、多备份](#Block的压缩、多备份)
  1. [BlockMaster对集群变动的管理](#BlockMaster对集群变动的管理)
  1. [负载均衡](#负载均衡)
1. [重构：引入NIO做数据传输](#重构：引入NIO做数据传输)
  1. [从Block支持NIO开始](#从Block支持NIO开始)
  1. [设计模式：NIO、非NIO和谐共处](#设计模式：NIO、非NIO和谐共处)
  1. [对client端而言，NIO只是一种可选的传输方式](#对client端而言，NIO只是一种可选的传输方式)
1. [NameServer的设计与实现](#NameServer的设计与实现)
  1. [变相偷懒：要学会假设](#变相偷懒：要学会假设)
  1. [NameServer的设计与实现](#NameServer的设计与实现)
  1. [偷懒变成了高级特性](#偷懒变成了高级特性)
1. [NameServer:一个完整的分布式系统出炉了](#NameServer:一个完整的分布式系统出炉了)
  1. [功能测试：release前捕杀bug的最后机会](#功能测试：release前捕杀bug的最后机会)
  1. [设计并编写benchmark](#设计并编写benchmark)
  1. [bug无法穷尽，良好的设计和优秀的代码质量才是王道](#bug无法穷尽，良好的设计和优秀的代码质量才是王道)
  1. [这个分布式系统能用吗](#这个分布式系统能用吗)

***

<a name="DistributeFS的概要设计"></a>
DistributeFS的概要设计
----------------------------------
<a name="一个简单到不能再简单的概要设计"></a>
###一个简单到不能再简单的概要设计
先根据需求给出一个概要设计，从满足需求的角度出发，参考一些其他的系统实现。

<a name="系统的组成模块"></a>
###系统的组成模块
BlockServer、BlockMaster、NameServer。
给出整体的框图就够了，概要设计为了证明设计的有效性，并不用来指导编码。我们幸运的是，整个系统的各个模块都是独立的，定义好interface，就可以分别做设计了。

<a name="FSClient的设计"></a>
###FSClient的设计
从client端的角度去做文件的增删改查，走通整个设计。（给出流程图）

<a name="开发环境"></a>
开发环境/技术依赖
----------------------------------
工欲善其事，必先利其器。

git，eclispe，java，ant，ivy，junit，contiperf，thrift

<a name="interfaces"></a>
interfaces
----------------------------------
<a name="Block是基本的数据结构"></a>
###Block是基本的数据结构
定义好一个block，接下来才会有block server和block master

<a name="BlockServer与BlockMaster的通讯接口"></a>
###BlockServer与BlockMaster的通讯接口
BlockServer需要与BlockMaster交换Block信息。

<a name="BlockMaster的服务接口"></a>
###BlockMaster的服务接口
Client端需要从BlockMaster获取Block的位置信息。然后据此找相应的BlockServer读写数据。

<a name="BlockServer的服务接口"></a>
###BlockServer的服务接口
BlockServer需要为Client提供数据的读写服务。

<a name="Block的第一个版本的实现"></a>
Block的第一个版本的实现
--------------------------------------
<a name="先写单元测试"></a>
###先写单元测试
一个简单的TestCase扯出来了一个通用的测试基类

Block的实现得用到input stream和output stream，需要在TestCase里面实现两个mock的stream。这两个stream具有通用性，那么我们就为TestCase提出一个基类，把stream放进去。

<a name="编码是一件愉快的事情"></a>
###编码是一件愉快的事情
对于非常明确的设计，实现一定要写的漂亮点。

<a name="对代码风格的理解"></a>
###对代码风格的理解
整洁的代码是最好的注释。

<a name="Block是必须是独立完美的"></a>
###Block是必须是独立完美的
对于整个系统来说，Block是一个原子性的类型，除了暴露出来的接口供BlockServer调用，我们可以对它一无所知。但是进入Block内部，它却是一个完整的、复杂的个体，有数据、index，内存索引等等。这就是封装。

<a name="BlockServer：容器与IO"></a>
BlockServer：容器与IO
----------------------------------------
<a name="BlockServer:Block的容器"></a>
###BlockServer:Block的容器
一个BlockServer管理多个block。

<a name="BlockServer:数据的存取服务"></a>
###BlockServer:数据的存取服务
数据的读写：先通过BlockServer找到Block，然后对Block中的某部分数据做读写访问。

<a name="设计模式让程序迸发智慧的光芒"></a>
###设计模式让程序迸发智慧的光芒
盘点BlockServer中用到的设计模式。

<a name="实现第一个版本的FSClient"></a>
实现第一个版本的FSClient
----------------------------------------
<a name="呈现给用户的接口一定要简单易用"></a>
###呈现给用户的接口一定要简单易用
除此之外，要屏蔽实现细节，最好把用户当成傻子。对用户的过多的期望和假设，只会让自己麻烦不断。

<a name="越过障碍：mock一个BlockMaster"></a>
###越过障碍：mock一个BlockMaster
没有BlockMaster，client还是跑不起来。这件事情好办，为了能尽快跑起来，我们mock一个BlockMaster。

<a name="第一次Server、Client联调"></a>
###第一次Server、Client联调


<a name="BlockMaster的实现"></a>
BlockMaster的实现
----------------------------------------
<a name="BlockServer的详细设计"></a>
###BlockServer的详细设计
BlockServer的详细设计，数据结构、消息同步、并发访问、异常处理、SPOF等等的都来了。

<a name="BlockServer数据结构设计"></a>
###BlockServer数据结构设计
Block信息的存放；Block与BlockServer的映射关系及索引；

<a name="坚守代码质量"></a>
###坚守代码质量
在到了最艰难的时刻，一定要坚守代码的质量，一定不要屈从于尽快搞定的快感，而写一些ugly的代码。

<a name="设计BlockServer备份节点"></a>
###设计BlockServer备份节点
有单点的系统不是真正的分布式系统，所以一定要消除单点。那就为BlockServer设计备份节点吧...

<a name="联调"></a>
联调
----------------------------------------
<a name="用BlockMaster替掉mock的那个"></a>
###用BlockMaster替掉mock的那个
基本组件已经全部完成了。

<a name="第二次Server、Client联调"></a>
###第二次Server、Client联调


<a name="分布式系统初步可用"></a>
###分布式系统初步可用
我们的分布式系统已经可以用了诶

<a name="补课"></a>
补课
----------------------------------------

<a name="config是一个重要的模块，而不是一个static final域"></a>
###config是一个重要的模块，而不是一个static final域

<a name="Block的压缩、多备份"></a>
###Block的压缩、多备份

<a name="BlockMaster对集群变动的管理"></a>
###BlockMaster对集群变动的管理

<a name="负载均衡"></a>
###负载均衡

<a name="重构：引入NIO做数据传输"></a>
重构：引入NIO做数据传输
-----------------------------------------
<a name="从Block支持NIO开始"></a>
###从Block支持NIO开始

<a name="设计模式：NIO、非NIO和谐共处"></a>
###设计模式：NIO、非NIO和谐共处

<a name="对client端而言，NIO只是一种可选的传输方式"></a>
###对client端而言，NIO只是一种可选的传输方式
Server端已是天翻地覆；对客户端接口而言，变化的只是数据传输时的一个不起眼的参数而已。“对你好而不让你知道”，对于一个分布式系统的码农而言，这是一种最酷的感情表达方式。


<a name="NameServer的设计与实现"></a>
NameServer的设计与实现
------------------------------------------
<a name="变相偷懒：要学会假设"></a>
###变相偷懒：要学会假设

<a name="NameServer的设计与实现"></a>
###NameServer的设计与实现

<a name="偷懒变成了高级特性"></a>
###偷懒变成了高级特性
系统中多了一个可插拔的模块

<a name="NameServer:一个完整的分布式系统出炉了"></a>
一个完整的分布式系统出炉了
------------------------------------------
<a name="功能测试：release前捕杀bug的最后机会"></a>
###功能测试：release前捕杀bug的最后机会

<a name="设计并编写benchmark"></a>
###设计并编写benchmark

<a name="bug无法穷尽，良好的设计和优秀的代码质量才是王道"></a>
###bug无法穷尽，良好的设计和优秀的代码质量才是王道

<a name="这个分布式系统能用吗"></a>
###这个分布式系统能用吗
这个我真的不知道，你才是勇敢的先行者。



