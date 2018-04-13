# md_blockchain
Java区块链平台，基于Springboot开发的区块链平台。区块链qq交流群737858576，刚建的群，一起学习区块链平台开发。
### 起因

公司要开发区块链，原本是想着使用以太坊开发个合约或者是使用个第三方平台来做，后来发现都不符合业务需求。原因很简单，以太坊、超级账本等平台都是做共享账本的，有代币和挖矿等模块。而我们需要的就是数家公司组个联盟，来共同见证、记录一些不可篡改的交互信息，如A公司给B公司发了一个xxx请求，B公司响应了什么什么。其实要的就是一个分布式数据库，而且性能要好，不能像比特币那种10分钟才生成一个区块。我们要的更多的是数据库的性能，和区块链的一些特性。

### 经过

项目于3月初开始研发，历时一月发布了第一版。主要做了存储模块、加密模块、网络通信、公钥私钥、区块内容解析落地入库等。已经初步具备了区块链的基本特征，但在共识机制、merkle tree、智能合约以及其他的一些细节上，尚不到位。

希望高手不吝赐教，集思广益，提出见解或方案，来做一个区块链平台项目，适合更多的区块链场景，而不仅仅是账本和各种忽悠人的代币。

理想中的区块链平台：
![输入图片说明](https://img-blog.csdn.net/2018021113252218 "在这里输入图片标题")

### 项目说明
主要有存储模块、网络模块、加密模块、区块解析入库等。

该项目属于"链"，非"币"。不涉及虚拟币和挖矿。本质上类似于腾讯区块链项目trustsql。

### 存储模块
Block内存储的是类Sql语句。联盟间预先设定好符合业务场景需要的数据库表结构，然后设定好各个节点对表的操作权限（ADD，UPDATE，DELETE），将来各个节点就可以按照自己被允许的权限，进行Sql语句的编写，并打包至Block中，再全网广播，等待全网校验签名、权限等信息的合法性。如果Block合法，则允许生成，生成后再全网广播，各节点拉取新区块。新区块生成后，各节点进行区块内容解析，并落地入库的操作。

场景就比较广泛了，可以设定不同的表结构，或者多个表，进而能完成各自类型信息的存储。譬如商品溯源，从生产商、运输、经销商、消费者等，每个环节都可以对某个商品进行ADD信息的操作。

存储采用的是key-value数据库rocksDB，了解比特币的知道，比特币用的是levelDB，都是类似的东西。最近发现在部分Windows下，rocksDB加载失败。也可以替换为levelDB，只需要修改2个类即可。

结构类似于sql的语句，如ADD（增删改） tableName（表名）ID（主键） JSON（该记录的json）。

### 网络模块
网络层，采用的是各节点互相长连接、断线重连，然后维持心跳包。网络框架使用的是t-io，也是oschina的知名开源项目。t-io采用了AIO的方式，在大量长连接情况下性能优异，资源占用也很少，并且具备group功能，特别适合于做多个联盟链的SaaS平台。并且包含了心跳包、断线重连、retry等优秀功能。

在项目中，每个节点即是server，又是client，作为server则被其他的N-1个节点连接，作为client则去连接其他N-1个节点的server。同一个联盟，设定一个Group，每次发消息，直接调用sendGroup方法即可。

### 共识模块
分布式共识一直都是个难题。

比特币采用了POW工作量证明，需要耗费大量的资源进行hash运算（挖矿），由矿工来完成生成Block的权利。还有一些是采用选举投票的方式来决定谁来生成Block。分布式共识算法有好几种，具体的可以去查查。共同的特点就是只能特定的节点来生成区块，然后广播给其他人。

而我这里的场景不同，这是一个联盟，各个节点是平等的，而且性能要高。所以我不想让每个节点都生成一个指令后，发给其他节点，再大家选举出一个节点来搜集网络上的指令组合再生成Block，太复杂了。对于学习区块链的新手来说，几乎无法完成这个模块。

那么我这里就简单多了，任何节点都可以构建Block，然后全网广播，只要过半节点校验后同意即可。其他节点需要校验格式、hash、签名、和table的权限，校验通过后，过半同意了，就可以构建Block并广播全网，通知各节点更新Block。

生成Block后，再全网广播，拉取Block，然后执行Block内的sql语句。

### 区块信息查询

各节点通过执行相同的sql来实现一个同步的sqlite数据库（或mysql等其他关系型数据库），将来对数据的查询都是直接查询sqlite，性能高于传统的区块链项目。

由于各个节点都能生成Block，在高并发下会出现区块不一致的情况。如果因为某些原因导致链分叉了，也提供了回滚机制，sql可以回滚。原理也很简单，你ADD一个数据时，我会在区块里同时记录两个指令，一个是ADD，一个是回滚用的DELETE。同理，UPDATE时也会保存原来的旧数据。区块里的sql落地，譬如顺序执行1-10个指令，回滚时就是从10-1执行回滚指令。

每个节点都会记录自己已经同步了的区块的值，以便随时进行sql落地入库。

对区块链信息的查询，那就简单了，直接做数据库查询即可。相比于比特币需要检索整个区块链的索引树，速度和方便性就大不同了。

### 简单使用说明

使用方法：先启动[md_blockchain_manager项目](https://gitee.com/tianyalei/md_blockchain_manager，)，然后修改application.yml里的name、appid和managerUrl和manager项目数据库里的一一对应，作为一个节点启动即可。

可以通过访问localhost:8080/block?content=1来生成一个区块，至少要启动2个节点才行，生成Block时需要除自己外的至少过半同意才行。生成Block后就会发现别的节点也会自动同步自己新生成的Block。目前代码里默认设置了一张表message，里面也只有一个字段content，相当于一个简单的区块链记事本了。

可以通过localhost:8080/sqlite来查看sqlite里存的数据，就是根据Block里的sql语句执行后的结果。

我把项目部署到docker里了，共启动4个节点，如图：
![输入图片说明](https://gitee.com/uploads/images/2018/0404/105151_c8931604_303698.png "1.png")

manager就是md_blockchain_manager项目，主要功能就是提供联盟链内各节点ip
![输入图片说明](https://gitee.com/uploads/images/2018/0404/105409_5e24cb3a_303698.png "1.png")

四个节点ip都写死了，都启动后，它们会相互全部连接起来，并维持住长连接和心跳包。
![输入图片说明](https://gitee.com/uploads/images/2018/0404/105748_bc6896d8_303698.png "1.png")

我调用一下block项目的生成区块接口，http://ip:port/block?content=1

![输入图片说明](https://gitee.com/uploads/images/2018/0404/105945_9e7f946f_303698.png "1.png")

别的节点会是这样，收到block项目请求生成区块的请求、并开始校验，回复是否同意
![输入图片说明](https://gitee.com/uploads/images/2018/0404/110142_cae21d7f_303698.png "1.png")

当block项目收到过半的同意后，就开始生成区块，并广播给其他节点自己的新区块，其他节点开始拉取新块，校验通过了则更新到本地。

这个生成区块的接口是写好用来测试的，正常走的流程是调用instuction接口，先生产符合自己需求的指令，然后组合多个指令，调用BlockController里的生成区块接口。


