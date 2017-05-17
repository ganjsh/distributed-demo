﻿Fourinone
=========

四合一分布式计算框架

淘宝Fourinone（中文名字“四不像”）是一个四合一分布式计算框架，在写这个框架之前，我对分布式计算进行了长时间的思考，也看了老外写的其他开源框架,当我们把复杂的hadoop当作一门学科学习时，似乎忘记了我们想解决问题的初衷：我们仅仅是想写个程序把几台甚至更多的机器一起用起来计算，把更多的cpu和内存利用上，来解决我们数量大和计算复杂的问题，当然这个过程中要考虑到分布式的协同和故障处理。如果仅仅是为了实现这个简单的初衷，为什么一切会那么复杂，我觉的自己可以写一个更简单的东西，它不需要过度设计，只需要看上去更酷一点，更小巧一点，功能更强一点。于是我将自己对分布式的理解融入到这个框架中，考虑到底层实现技术的相似性，我将Hadoop,Zookeeper,MQ,分布式缓存四大主要的分布式计算功能合为一个框架内，对复杂的分布式计算应用进行了大量简化和归纳。

首先，对分布式协同方面，它实现了Zookeeper所有的功能，并且做了很多改进，包括简化Zookeeper的树型结构，用domain/node两层结构取代，简化Watch回调多线程等待编程模型，用更直观的容易保证业务逻辑完整性的内容变化事件以及状态轮循取代，Zookeeper只能存储信息不大于1M的内容，Fourinone超过1M的内容会以内存隐射文件存储，增强了它的存储功能，简化了Zookeeper的ACL权限功能，用更为程序员熟悉rw风格取代，简化了Zookeeper的临时节点和序列节点等类型，取代为在创建节点时是否指定保持心跳，心跳断掉时节点会自动删除。Fourinone是高可用的，没有单点问题，可以有任意多个复本，它的复制不是定时而是基于内容变更复制，有更高的性能，Fourinone实现了领导者选举算法（但不是Paxos），在领导者服务器宕机情况下，会自动不延时的将请求切换到备份服务器上，选举出新的领导者进行服务，这个过程中，心跳节点仍然能保持健壮的稳定性，迅速跟新的领导者保持心跳连接。基于Fourinone可以轻松实现分布式配置信息，集群管理，故障节点检测，分布式锁，以及淘宝configserver等等协同功能。

其次, Fourinone可以提供完整的分布式缓存功能。如果对一个中小型的互联网或者企业应用，仅仅利用domain/node进行k/v的存储即可，因为domain/node都是内存操作而且读写锁分离，同时拥有复制备份，完全满足缓存的高性能与可靠性。对于大型互联网应用，高峰访问量上百万的并发读写吞吐量，会超出单台服务器的承受力，Fourinone提供了fa?ade的解决方案去解决大集群的分布式缓存，利用硬件负载均衡路由到一组fa?ade服务器上，fa?ade可以自动为缓存内容生成key，并根据key准确找到散落在背后的缓存集群的具体哪台服务器，当缓存服务器的容量到达限制时，可以自由扩容，不需要成倍扩容，因为fa?ade的算法会登记服务器扩容时间版本，并将key智能的跟这个时间匹配，这样在扩容后还能准确找到之前分配到的服务器。另外，基于Fourinone可以轻松实现web应用的session功能，只需要将生成的key写入客户端cookie即可。

Fourinone对于分布式大数据量并行计算的解决方案不同于复杂的hadoop，它不像hadoop的中间计算结果依赖于hdfs，它使用不同于map/reduce的全新设计模式解决问题。Fourinone有“包工头”，“农民工”，“手工仓库”的几个核心概念。“农民工”为一个计算节点，可以部署在多个机器，它由开发者自由实现，计算时，“农民工”到“手工仓库”获取输入资源，再将计算结果放回“手工仓库”返回给“包工头”。“包工头”负责承包一个复杂项目的一部分，可以理解为一个分配任务和调度程序，它由开发者自己实现，开发者可以自由控制调度过程，比如按照“农民工”的数量将源数据切分成多少份，然后远程分配给“农民工”节点进行计算处理，它处理完的中间结果数据不限制保存在hdfs里，而可以自由控制保存在分布式缓存、数据库、分布式文件里。如果需要结果数据的合并，可以新建立一个“包工头”的任务分配进行完成。多个“包工头”之间进行责任链式处理。总的来说，是将大数据的复杂分布式计算，设计为一个链式的多“包工头”环节去处理，每个环节包括利用多台“农民工”机器进行并行计算，无论是拆分计算任务还是合并结果，都可以设计为一个单独的“包工头”环节。这样做的好处是，开发者有更大能力去深入控制并行计算的过程，去保持使用并行计算实现业务逻辑的完整性，而且对各种不同类型的并行计算场景也能灵活处理，不会因为某些特殊场景被map/reduce的框架限制住思维，并且链式的每个环节也方便进行监控过程。

Fourinone也可以当成简单的mq来使用，将domain视为mq队列，每个node为一个队列消息，监控domain的变化事件来获取队列消息。也可以将domain视为订阅主题，将每个订阅者注册到domain的node上，发布者将消息逐一更新每个node，订阅者监控每个属于自己的node的变化事件获取订阅消息，收到后删除内容等待下一个消息。但是Fourinone不实现JMS的规范，不提供JMS的消息确认和消息过滤等特殊功能，不过开发者可以基于Fourinone自己去扩充这些功能，包括mq集群，利用一个独立的domain/node建立队列或者主题的key隐射，再仿照上面分布式缓存的智能根据key定位服务器的做法实现集群管理。

Fourinone整体代码短小精悍，跟Hadoop, Zookeeper, Memcache, ActiveMq等开源产品代码上没有任何相似性，不需要任何依赖，引用一个jar包就可以嵌入式使用，良好支持window环境，可以在一台机器上模拟分布式环境，更方便开发。

开发包里自带了一系列傻瓜上手demo，包括分布式计算、统一配置管理、集群管理、分布式锁、分布式缓存、MQ等方面, 每个demo均控制在少许行代码内，但是涵盖了Fourinone主要的功能，方便大家快速理解并掌握。


Fourinone 2.0新增功能:
Fourinone2.0提供了一个4合1分布式框架和简单易用的编程api，实现对多台计算机cpu，内存，硬盘的统一利用，从而获取到强大计算能力去解决复杂问题。Fourinone框架提供了一系列并行计算模式（农民工/包工头/职介绍/手工仓库）用于利用多机多核cpu的计算能力；提供完整的分布式缓存和小型缓存用于利用多机内存能力；提供像操作本地文件一样操作远程文件（访问，并行读写，拆分，排它，复制，解析，事务等）用于利用多机硬盘存储能力；由于多计算机物理上独立，Fourinone框架也提供完整的分布式协同和锁以及简化MQ功能，用于实现多机的协作和通讯。

一、提供了对分布式文件的便利操作, 将集群中所有机器的硬盘资源利用起来，通过统一的fttp文件路径访问，如:
windows：fttp://192.168.0.1/d:/data/a.log
linux：fttp://192.168.0.1/home/user/a.log

比如以这样的方式读取远程文件：
FttpAdapter fa = FttpAdapter("fttp://192.168.0.1/home/log/a.log");
fa.getFttpReader().readAll();

提供对集群文件的操作支持，包括：
1、元数据访问，添加删除，按块拆分, 高性能并行读写，排他读写（按文件部分内容锁定），随机读写，集群复制等
2、对集群文件的解析支持（包括按行，按分割符，按最后标识读取）
3、对整形数据的高性能读写支持（ArrayInt比ArrayList存的更多更快）
4、两阶段提交和事务补偿处理
5、自带一个集群文件浏览器，可以查看集群所有硬盘上的文件（不同于hadoop的namenode,没有单点问题和容量限制）

总的来说, 将集群看做一个操作系统，像操作本地文件一样操作远程文件。
但是fourinone并不提供一个分布式存储系统，比如文件数据的导入导出、拆分存储、负载均衡，备份容灾等存储功能，不过开发人员可以利用这些api去设计和实现这些功能，用来满足自己的特定需求。

二、自动化class和jar包部署
class和jar包只需放在工头机器上， 各工人机器会自动获取并执行，兼容操作系统，不需要进行安全密钥复杂配置

三、网络波动状况下的策略处理，设置抢救期，抢救期内网络稳定下来不判定结点死亡


fourinone-3.04.25最新版升级内容:
1、编译和运行环境升级为jdk7.0版本；
2、计算中止和超时中止的支持，比如多台计算机工人同时执行查找，一旦某台计算机工人找到，其余工人全部中止并返回。以及可以由工人控制或者框架控制的计算过程超时中止。
3、一次性启动多工人进程支持，可以通过程序api一次性启动和管理“ParkServer/工头/工人”多个进程，并附带良好的日志输出功能，用于代替写批处理脚本方式，方便部署和运行。
4、工人服务化模式的支持，把工人当作通用服务来使用和封装。
5、增加了相应指南和demo。

Fourinone4.0版新特性：一个高性能的数据库引擎CoolHash(酷哈嘻)
1、CoolHash是一个数据库引擎
CoolHash只做数据库最基础核心的引擎部分，支持大部分数据类型的“插入、获取、更新、删除、批量插入、批量获取、批量更新、批量删除、高效查询（精确查、模糊查、按树节点查、按key查、按value查）、分页，排序、and操作、or操作、事务处理、key指针插入和查询、缓存持久互转”等操作和远程操作。

其他的“监控、管理、安全、备份、命令行操作、运维工具”等外围特性都剥离出去不做，开发者可以根据自己需求扩充这些功能，CoolHash也不做自动扩容，CoolHash认为分布式集群特性也可以在外围通过“分库分表”或者“分布式扩容”等中间件技术去做，目前国内很多企业都具备基于开源软件做外围中间件的能力，这样，CoolHash只维持一个高性能又轻量级的最小存储引擎。

CoolHash高度产品化，易用性强，容易嵌入使用和复制传播（200k大小），采用apache2.0开源协议，使用java实现(jdk1.7)，对外提供java接口，同时支持windows和linux（unix-like），由于依赖底层操作系统，windows和linux的实现稍有不同。

2、CoolHash是一个k/v数据库
实现数据库存储结构索引有多种方法，有比较平衡减少深入的b树、b+树系列，结合内存再合并的LSM-tree系列（bigTable），借鉴字典索引技术的Trie树（前缀树、三叉树）等，不过对于key/value结构，感觉最合适的还是Hash，不过java里的Hash算法是实现在内存组件里，无法持久化，只能快速读写，但是无法模糊查询，传统的Hash不是一个“cool”的Hash，需要进行改进。

CoolHash改进后的Hash算法是一个完整的key树结构，我们知道传统Hash的key和key之间没有关系，互相独立，但是在CoolHash里，key可以表示为“user.001.name”的用“.”分开的树层次结构，如“user.001.age”和“user.001.name”都属于“user.001.*”分支，既可独立获取，也可以从父节点查询，还可以“user.*.name”方式只查询所有子节点。

提出key指针的概念：CoolHash的key可以是一个指针，指向其他树的key，这样能将两棵key树连起来，这样的设计能避免大量join操作，如果两棵key树没有直接关系，需要动态join将会非常耗时，但有了key指针可以很好的描述数据之间“1对1、1对多、多对多”的关联关系。key指针也不同于数据库外健，它可以模糊指，比如只模糊指向一个key前缀“user.001”，再补充需要获取的属性形成一个完整key；还能连续指，比如一个key指针指向另一个key指针再指向其他key指针，连续指可以将更多的树联系起来形成一个大的数据森林。CoolHash能很好的控制key指针最终值返回、key指针死循环、读写死锁等问题。

我们知道关系数据库的数据表是一个行列的二维矩阵的数据结构，表和表之间没有层次感，一个表不会是另一个表的父子节点，这就导致需要大量关联，从一个表获取另一个表的数据需要通过join操作，同时由于表和表之间彼此独立，又导致数据量大后join性能不好，多个矩阵需要动态的“求并”“求或”获取主健，再返回所有属性。很多朋友是开发业务系统出身，复杂一点的业务逻辑通常需要关联7-8个数据库表甚至更多，是相当头疼的事情。

这种矩阵行列式结构还有个不好，就是一加全加，一减全减，一行加了一列属性，所有行都要跟着加，假如有个“子女”属性，有的人有1个子女，有的人有2-3个子女，这个属性应该有时是1列，有时是3列，有时没有才好。CoolHash的父节点下的key值可以任意添加和减少，或者再任意向下扩充叶子结点，也可以任意查询，能够非常灵活，比如“子女”属性可以这样表示第一个小孩：“user.001.children.001”。

关系数据库还容易产生数据碎片，需要经常执行“OPTIMIZE TABLE”或者“压缩数据库”等操作回收空间，CoolHash对此做了设计和算法改进，对于大量数据频繁写入和删除能很好重复利用存储空间。

key和value的数据格式：CoolHash的key只能是字符串，默认最大长度为256字节。value能支持非常广泛的数据类型，基本数据类型“String（变长字符）、short（短整形）、int（整型）、long（长整形）、 double（双精度浮点型）、 float（浮点型）、Date（日期型）”，高级数据类型的大部分的java集合都能支持（List、Map、Set等），以及任意可序列化的自定义java类型，底层数据类型也可以支持二进制型。基本数据类型的value可支持按内容查询，高级数据类型和二进制类型不支持按内容查询。所有类型的value默认最大长度为2M，默认配置保证一对key/value长度不会太大，能够容易加载到内存进行处理，但是key和value的最大长度、以及region大小都是可以根据计算机性能进行配置。

CoolHash还支持HashMap缓存和持久化的统一和互相转换，并共用一套API，可以将数据放入CoolHashMap，也可以放入持久化，可以将CoolHashMap对象转成持久化，也可以从持久化数据里直接获取CoolHashMap对象。

3、CoolHash是一个并行数据库（mpp）
前几年有篇文章，国外的数据库大牛猛烈抨击map/reduce，让重复造轮子的人应该去学习一下关系数据库几十年的理论和实践积累，一个借助蛮力而不善于设计索引的数据库系统是愚蠢和低效的。这一方面说明了map/reduce技术已经侵犯到了传统数据库领域的核心利益，另一方面也暴露了分布式存储技术的某些不足。

这里的蛮力就是并行计算，在CoolHash的底层，会维持一个数据工人进程组，根据计算机性能少则几个多则上百个，对于数据库的每种操作，数据工人们像演奏交响乐一样，时而独奏，时而合奏，统一调度，紧密协作的完成任务，CoolHash从一开始就是高度并行化的设计，数据库引擎本质就是在寻求cpu、内存、硬盘的充分利用和均衡利用，“一力胜十巧”，在大量数据的读写和查询中并行计算的效果犹为明显。

同时一个好的数据库引擎应该是蛮力和技巧的结合，Hbase的一个重要启示，就是可以灵活设计它的key达到近似索引的效果，CoolHash将此特性发挥的更深入，树型结构的key本身就是最好的索引，除外CoolHash几乎不需要另外再建立索引，只需要按照业务数据特点设计好你的key。并行计算和索引的结合会得到一个很好的互补，让你的索引结构不需要维持的太精准而节省开销，举个例子：我们从一个城市找一个人，一种方法是我们有精确的索引，知道他在哪个区哪个楼那个房间，另一种是我们只大致知道他在哪栋楼，但是我们有几百个工人可以每间房同时去找，这样也能很快找到。

4、CoolHash是一个nosql数据库
nosql不等于没有sql的功能，关系数据库的sql仍然是非常方便的交互语言，而且有专门的标准，CoolHash通过函数方式实现大部分sql的功能，如果需要扩充sql支持，在外围用正则表达式做一个sql解析，然后调用CoolHash的函数支持即可。

由于CoolHash没有关系数据库的“db、table、row、col”等概念，使用树型key取代了，所以sql create语句功能就不需要了。
“put()函数”对应于“sql insert、update”语句
“remove()函数”对应于“sql delete...where”语句
“put(map)函数”对应于“sql insert/update into...select...”语句
“get()函数”对应于“sql select...where id=?”语句
“get(map)”函数对应于“sql select *...”语句
“find(*,filter)”函数对于于“sql select * where col=%x%”语句
“and()函数”对应于“sql and”语句
“or()函数”对应于“sql or”语句
“sort(comp)函数”对应于“sql desc/group”语句


5、CoolHash实现了事务处理
CoolHash实现了ACID事务属性，对写入、更新、删除的基本操作提供事务处理，在程序里调用begin()、commit()、rollback()等事务方法。
原子性(Atomic)：事务内操作要么提交全部生效，要么回滚全部撤消。
一致性(Consistent)：事务操作前后的数据状态保持一致，一致性跟原子性密切相关，比如银行转账前后，两个账户累加总和保持一致。
隔离性(Isolated)：多个事务操作时，互相不能有影响，保持隔离。
持续性(Durable) ：事务提交后需要持久化生效。
另外，对于多个事务并发操作数据的情况，JDBC规范归纳出“脏读(dirty read)、不可重复读(non-repeatable read) 、幻读(phantom read) ”三种问题，并提出了4种事务隔离级让软件制造商去实现，按照不同等级去容忍这三个问题，其实这种逐级容忍大部分都不实用，一个事务操作未提交时，按照事务隔离级，其他访问应该是读取到该事务开始前的数据，而不应该把事情搞复杂，CoolHash实现的是“TRANSACTION_SERIALIZABLE”，禁止容忍三种读取问题。

6、CoolHash是一个数据库Server
CoolHash支持远程网络访问，服务端发布一个IP和监听端口，客户端连接该IP端口即可进行远程操作，CoolHash可承受多用户高并发的网络连接访问，来源于服务端设计的相似之处，大家知道Apache HTTP Server是一个多进程模型+共享内存方式实现，在前面讲到的CoolHash会维持一个数据工人的进程组，数据工人不仅是并行计算的执行者，同时也是网络请求的响应者，数据工人身兼多职能很好的将服务端设计统一起来，避免重复设计，因此CoolHash也是一个很好的网络Server.

7、CoolHash的测试性能
有句话叫做“人生如白驹过隙”，用来形容性能就是一瞬间，数据库性能最好能接近缓存，或者直接可以当作缓存来用，能在瞬间完成读写和各种查询，这个瞬间就是秒。只有在几秒内完成操作才能做到实时交互没有等待感，否则就要离线交互。
CoolHash的单条写入和读取速度都在毫秒级别，写和读差别不大，读略快于写。
CoolHash的批量写入和读取速度都控制在秒级别，100万数据写入基准测试，普通台式机或者笔记本（4核4g）需要5-6秒，标准pc server(24核256g)需要2-3秒；批量读、批量删除和批量写入的速度差不多。
CoolHash写入缓存和写入持久的速度差别不大，100万数据写入缓存基准测试，普通台式机或者笔记本（4核4g）需要5秒左右，标准pc server(24核256g)需要2秒左右。如果是10万级别的数据读写，缓存和持久的速度大致接近等同。
CoolHash的查询速度控制在秒级别，100万数据的模糊查询（如like%str%）在没有构建索引情况下，普通台式机或者笔记本（4核4g）需要2-3秒，标准pc server(24核256g)需要1-2秒；如果是重复查询，由于CoolHash内部做了数据内存映射，第二次以后只需要毫秒级完成。
高并发多客户端的吞吐量总体速度要快于单客户端，但是受服务器cpu、内存、io等性能限制，会倾向于一个平衡值。

数据库引擎的性能通常也会受“key/value数据大小、数据类型、工人数量、硬件配置（内存大小、cpu核数、硬盘io、网络耗用）”等等因素影响。
比如同样数量但是单条key/value数据很大，整体速度要慢一些；
基本数据类型的读写速度要快过高级数据类型（如java集合类）；
工人数量也有影响，对于单条写入读取单工人和多工人差不多，但是批量操作和查询多工人要好过单工人，普通台式机或者笔记本（4核4g）维持在8-10个工人可以打满cpu，标准pc server(24核256g)最大可以维持到100个左右工人，但是工人数量就算能打满cpu，也受限于后端硬盘io，到一定程度速度不再增长；
硬件配置对于性能的影响很大，普通笔记本的测试效果明显不如标准pc server，笔记本内存较小，硬盘io弱，不适合做数据库服务器。内存大的服务器测试效果好，加载数据和jvm垃圾回收速度会更快，目前测试采用的都是传统SAS/SATA硬盘，如果采用固态硬盘进行硬件升级，随机IO性能会得到进一步提升；
另外，由于存在网络耗用和序列化传送，远程网络操作的速度要比本地速度慢，但是局域网内速度会接近本地速度，这是因为远程网络操作涉及带宽接入限制和线路共享等复杂消耗，而局域网内主要取决于物理设备速度。

关于CoolHash性能的更多体验，欢迎有兴趣的朋友根据demo在各自的机器环境下，模拟各种极端条件下去压测。

三、如何使用CoolHash
CoolHash追求极简的编程体验，不需要安排配置，服务端启动CoolHashServer，指定好ip和端口，
客户端大致编程步骤如下：
CoolHashClient chc = BeanContext.getCoolHashClient("localhost",2014);//连接CoolHashServer
chc.put("user.001.name","zhang");//写入字符
chc.put("user.001.age",20);//写入整数
chc.put("user.001.weight",50.55f);//写入浮点数
chc.put("user.001.pet",new ArrayList());//写入集合对象

String name = (String)chc.get("user.001.name");//读取字符
int age = (int)chc.get("user.001.age");//读取整数
float weight = (float)chc.get("user.001.weight");//读取浮点数
ArrayList pet = (ArrayList)chc.get("user.001.pet");//读取集合对象

chc.put("user.002.name","Li");
chc.put("user.002.age",25);
chc.put("user.002.weight",60.55f);

CoolKeyResult keyresult = chc.findKey("user.001.*");//查找用户001的所有属性
CoolKeySet ks = keyresult.nextBatchKey(4);//分页获取前4条结果
System.out.println(ks);//输出[user.001.weight, user.001.age, user.001.name, user.001.pet]

CoolHashResult mapresult = chc.find("user.*.age", ValueFilter.greater(18));//查找年龄大于18岁的用户
CoolHashMap hm = mapresult.nextBatch(10);
System.out.println(hm);//输出[user.001.age=20, user.002.age=25]	
......
更多的功能使用请去参考开发包里的demo


除外，4.0版本还增加了以下特性：
1、多进程多线程的无缝融合，同一套接口，改改参数，从多进程变为多线程，开发者无需改写程序逻辑；
2、提供高容错任务分配算法API：doTaskCompete（m工人,n任务），将n个任务分给m个工人并行完成，根据任务大小设置工人数量，工人间能者多劳，性能好的工人机器争抢干更多的任务，同时跟现实工作一样，如果有工人生病请假（故障），那么他的任务活由其余工人代干，除非所有工人出故障，否则就算只剩一个工人也应该加班把其他所有工人的活干完，对整体计算来说，部分工人故障对计算结果来说不受影响，只是计算时间会延长。


本软件遵循apache2.0开源软件协议，自由分享
(C) 2007-2012 Alibaba Group Holding Limited

邮箱:Fourinone@yeah.net
qq群1:1313859
qq群2:241116021
qq群3:23321760

关于Fourinone的所有架构、指南、demo，可以参考《大规模分布式系统架构与设计实战》一书