需求和概要设计
==========
#需求描述
##一句话描述
小文件存储系统，分布式可伸缩的，支持海量数据、海量文件的安全存储，以及增删改查操作，系统设计侧重于CAP理论的CA特性。

##需求来源
在我们有道云笔记项目的开发过程中，存在对小文件存储系统的需求，不过因为有道没有现成的系统，从而使用了网易杭研院公司的一款产品。这中间的一个插曲是，我们自己也想开发个类似的系统，并且做了一些工作，但是因为采用了杭研院的系统，我们就没再造轮子。而我自己就用业余时间自己来造轮子，我造轮子的原因，不是需求驱动，而是兴趣所致，不以金钱衡量，所以造轮子也是有意义的。

下面说一下小文件存储系统的定位。在我们公司有一套类似于hadoop的存储计算系统，但是还是无法解决海量小文件的存储问题。用hadoop来描述该问题：hdfs作为GFS的开源实现，能够存储海量数据，但是因为文件索引和meta信息由NameNode集中存放和管理，导致文件总数受NameNode的内存限制，无法存放大量的小文件，反而适合放比较大的文件；再者，NameNode单点问题（single point of failure, SPOF）的解决是一个非常繁杂的问题，hadoop也是在2.0.0版本才解决这个问题，解决这个问题的部分逻辑甚至都可以独立成一个工程：BookKeeper。而HBase,作为一个BigTable的开源社区的实现，能够存储海量的key；但作为一个数据库，不提供流式的数据访问，单个row如果太大，比如大于1M，使用rpc读写数据，性能会很差；此外BigTable的结构决定了其磁盘内存使用比比较小，内存开销太大，不适合做文件系统。综上，小文件存储系统定位是介于GFS与BigTable之间的，能够存海量的、大小不一的数据。

#现有文件系统分析
下面简单的分析和列举一些现有的文件系统及其概要的设计，集思广议。
**MooseFS**
: 与HDFS类似，是基于GFS架构的分布式文件系统，目前还没解决Master节点的SPOF问题。

**HayStack**
: HayStack 是facebook的图片存储系统，论文见[Finding a needle in Haystack: Facebook’s photo storage][haystack_Link]。出于图片的一次写多次读，并且存在热点数据（越晚写的数据读的越频繁）的特性，HayStack做了特殊的设计，将批量数据及其meta信息拼接成一个大文件（Superblock
）并为其建立index file，index file常驻内存。数据的删除只是对index file做标记。可以对Superblock做压缩和合并，以去除重复和已删除的数据。HayStack使用Directory Server来解析请求，找到制定的Store的Superblock上，然后通过index文件找到数据所在位置并读取。HayStack系统还包括Cache Server和CDN，以优化系统吞吐性能。HayStack实现的需求可能与我们做的小文件系统很类似，尤其是SuperBlock的设计，有很强的参照性。

**Google MegaStore**
: Google MegaStore是在BigTable的基础上做了一层封装，支持RDBMS的特性，如支持sql，分布式事务。本质上，是一个数据库系统，而非文件存储系统。

**Google Spanner**
: google新开发出来的神器，只看了论文，这是一个庞大的东西，涉及到大规模集群、海量数据、跨地域等概念，甚至需要借助卫星来同步时钟。这个东西的实现不是我们可以考虑的，也许可以期待几年之后开源社区的代码吧。

**MogileFS**
: 这是一个非常有趣的系统，看一些介绍，感觉和我们做的事情比较像，还没来得及详细了解，这里先mark一下，回头再来看。我迫不及待的要开始写自己的设计。

#DistributeFS的概要设计
##存储结构
存储结构应该仿照HayStack，由数据文件(BlockData)和数据索引文件（BlockDataIndex）组成，我们也不妨称之为SuperBlock。在这样的设计下，整个系统的SuperBlock数量应该是可控的。比如，我们可以考虑将每个SuperBlock大小控制在1G。那么存1P的数据只需要1M个SuperBlock，对SuperBlock的管理将会变得比较简单。写文件将是对SuperBlock及BlockDataIndex的appending操作；BlockDataIndex文件将常驻内存，读取时，通过BlockDataIndex得到文件在BlockData的offset及length，然后读取；删除操作是对BlockDataIndex做删除标记。SuperBlock将由BlockServer来负责存取。

##SuperBlock的管理
SuperBlock将由BlockMaster来管理，BlockMaster负责SuperBlock的索引服务及管理服务。当有新的写请求时，新建或选择适当的SuperBlock来写数据。读取数据时，告诉客户端文件所在的SuperBlock的位置，客户端找到相应的BlockServer服务器去读数据。此外，还需要对SuperBlock做备份管理，一个SuperBlock将存放在多个BlockServer上，如果有BlockServer下线，需要将受影响的SuperBlock从现存的BlockServer复制到新的BlockServer上。整个这个部分与HDFS对DataBlock的管理十分的类似。

##文件的描述符
文件的描述符将由两个部分组成，SuperBlockId以及DataIndexId。通过SuperBlockId找到文件所在的SuperBlock，然后通过DataIndexId从BlockDataIndex中找到文件的index，然后再由index从BlockData中读到真正的数据。

##BlockMaster的SPOF问题
通过上述描述，我们可以发现，系统是由一个BlockMaster和多个BlockServer组成的。BlockServer会存在单点失效问题。这是在后续开发中需要考虑的问题。因为BlockMaster还是一个非常笼统的概念，目前先不对其SPOF的解决方案做概要设计。但是在后续对BlockMaster的设计和开发中，需要时刻考虑每个动作对将来解决SPOF文件带来的困难。这里需要mark的一点是：做系统设计时，前瞻性的考虑和设计是非常有必要的，不然设计的思路会越来越窄，代码也会越来越脏。

##文件的目录服务
通过上面几个部分的设计，我们已经得到了一个分布式的文件系统，每写一个文件，可以给返回一个文件描述符，通过这个描述符可以得到文件数据。

如果说还缺少什么，那就是目录服务。如果用户想存一个文件，然后把这个文件记录成一个特殊的key或者path的话，我们的系统是无法满足需求的。需要用户将key或者path与文件描述符做映射并持久存储。下次访问时，先用key或path查找文件描述符，然后再读文件。因此，我们这里还需有一个目录服务（NameServer）来做上述事情。对于目录服务的设计是一件比较难的事情：如果是文件用key描述的话，我们实质上是提供持久存储且无SPOF问题的kv服务；如果文件用path来描述的话，我们提供的就是一个持久存储且无SPOF问题的目录树服务。

这个事情可能与HDFS的NameNode的SPOF问题一样的难以解决，这里先不考虑这些问题。很幸运的是，即使没有NameServer，我们也是一个分布式文件系统，NameServer与前面的设计可以相互独立。Mark:我做设计，非常喜欢的感觉就是能将逻辑或者结构彻底的拆解开，我喜欢的一个词：相互独立；在一个大而复杂的系统里，没有什么事情比解耦合来的更美妙了。

#开始编码了？
##再通读上面的设计
再回过头来读一下上面的设计。是的，这些想法在我的脑海里面已经盘旋了好久了，虽然很多地方细节是模糊的，但我认为整体设计是可行的。再考虑一下，可以做到对文件的"增删查"，那"改"怎么办？仔细想一想，改的情况可以分为很多：（1）相同文件名写一个完全新的文件；（2）中间位置插入或删除一段数据；(3)文件最后位置appending一段新数据。对于(1)非常好解决，写一个新文件，在NameServer里，用新的文件描述符替换掉当前的文件描述符；对于(2)目前看来不太好解决，不过影响不大，目前绝大部分分布式系统都不支持该操作；对于（3）这个可以使用文件指针来实现，这个应该有比较成熟的方案。
Mark，再做完一个设计开始编码之前，一定要对着需求看一遍设计。“是的，你的设计很美妙，但是到最后才发现，原来有个需求不好满足”，这是一件非常悲惨的设计，这样的设计有缺陷和硬伤，会毁掉你之前所有的努力。

##那就开始编码吧
编码从哪里开始？我认为应该从框架和抽象开始。写软件如同盖房子，好的设计师，在动土之前房子已经在脑海里了；好的程序员，写程序要先用抽象的接口和数据结构，把最基本的模块和实体给勾勒出来，然后再去不断的实现具体的功能。
说到底，从哪开始，我思索半天，应该先把BlockMaster和BlockServer对外提供的服务接口给描述出来。恩，那就开始写第一行代码吧。



[haystack_Link]: http://static.usenix.org/event/osdi10/tech/full_papers/Beaver.pdf "Finding a needle in Haystack: Facebook’s photo storage"
