# SolrCloud

## SolrCloud的基本概念
- Cluster集群：一组Solr节点，逻辑上作为一个单元进行管理，整个集群使用同一套Schema和SolrConfig

- Node节点：一个运行Solr的JVM实例

- Collection：在SolrCloud集群中逻辑意义上的完整的索引，常常被划分为一个或多个Shard。这些Shard使用相同的config set,如果Shard数超过一个，那么索引方案就是分布式索引。
- Core：也就是Solr Core，一个Solr中包含一个或者多个SolrCore，每个Solr Core可以独立提供索引和查询功能，Solr Core额提出是为了增加管理灵活性和共用资源。SolrCloud中使用的配置是在Zookeeper中的，而传统的Solr Core的配置文件是在磁盘上的配置目录中。

- Config Set:Solr Core提供服务必须的一组配置文件，每个Config Set有一个名字。必须包含solrconfig.xml和schema.xml，初次之外，依据这两个文件的配置内容，可能还需要包含其他文件。Config Set存储在Zookeeper中，可以重新上传或者使用upconfig命令进行更新，可以用Solr的启动参数bootstrap_confdir进行初始化或者更新。

- Shard分片：Collection的逻辑分片。每个Shard被分成一个或者多个replicas，通过选举确定那个是Leader。

- Replica：Shard的一个拷贝。每个Replica存在于Solr的一个Core中。

- Leader：赢得选举的Shard replicas，每个Shard有多个replicas，这几个Replicas需要选举确定一个Leader。选举可以发生在任何时间。当进行索引操作时，SolrCloud将索引操作请求传到此Shard对应的leader，leader再分发它们到全部Shard的replicas

![](https://raw.githubusercontent.com/zylele/solr-notes/master/screenshots/solr/logic.png)

在SolrCloud模式下，Collection是访问Cluster的入口。Collection是一个逻辑存在的东西，可以跨Node，在任意节点上都可以访问Collection；Shard也是逻辑存在的，因此Shard也是可以跨Node的；一个Shard下面可以包含0个或者多个replica，但1个Shard下面只能包含一个leader

![](https://raw.githubusercontent.com/zylele/solr-notes/master/screenshots/solr/work.png)

SolrCloud中包含有多个Solr Instance，而每个Solr Instance中包含有多个Solr Core，Solr Core对应着一个可访问的Solr索引资源Replica，当Solr Client通过Collection访问Solr集群的时候，便可以通过Shard分片找到对应的replica即SolrCore，从而就可以访问索引文档了

## 实例

起两个实例，指定zk及solr端口

`solr start -cloud -z 127.0.0.1:2181 -p 8983`
`solr start -cloud -z 127.0.0.1:2181 -p 8984`

创建集合
`solr create_collection -c user_info -shards 2 -replicationFactor 2`

下载配置集
`solr zk downconfig -z 127.0.0.1:2181 -n user_info -d path\user_info_config`

上传配置集
`solr zk upconfig -z 127.0.0.1:2181 -n user_info -d path\user_info_config`

reload
`{solr.host}/admin/collections?action=RELOAD&name=name`


## 不得不知道的索引存储细节

当Solr客户端发送add/update请求给CloudSolrServer,CloudSolrServer会连接至Zookeeper获取当前SolrCloud的集群状态，并会在/clusterstate.json 和/live_nodes中注册watcher，便于监视Zookeeper和SolrCloud，这样做的好处有以下两点：

1、CloudSolrServer获取到SolrCloud的状态后，它可直接将document发往SolrCloud的leader，从而降低网络转发消耗。

2、注册watcher有利于建索引时候的负载均衡，比如如果有个节点leader下线了，那么CloudSolrServer会立马得知，那它就会停止往已下线的leader发送document。

此外，CloudSolrServer 在发送document时候需要知道发往哪个shard？对于建好的SolrCloud集群，每一个shard都会有一个Hash区间，当Document进行update的时候，SolrCloud就会计算这个Document的Hash值，然后根据该值和shard的hash区间来判断这个document应该发往哪个shard，Solr使用documentroute组件来进行document的分发。目前Solr有两个DocRouter类的子类CompositeIdRouter(Solr默认采用的)类和ImplicitDocRouter类，当然我们也可以通过继承DocRouter来定制化我们的document route组件。

举例来说当Solr Shard建立时候，Solr会给每一个shard分配32bit的hash值的区间，比如SolrCloud有两个shard分别为A,B，那么A的hash值区间就为80000000-ffffffff，B的hash值区间为0-7fffffff。默认的CompositeIdRouter hash策略会根据document ID计算出唯一的Hash值，并判断该值在哪个shard的hash区间内。

SolrCloud对于Hash值的获取提出了以下两个要求：

1、hash计算速度必须快，因为hash计算是分布式建索引的第一步。

2、 hash值必须能均匀的分布于每一个shard，如果有一个shard的document数量大于另一个shard，那么在查询的时候前一个shard所花的时间就会大于后一个，SolrCloud的查询是先分后汇总的过程，也就是说最后每一个shard查询完毕才算完毕，所以SolrCloud的查询速度是由最慢的shard的查询速度决定的。

基于以上两点，SolrCloud采用了MurmurHash 算法以提高hash计算速度和hash值的均匀分布。

## Solr创建索引可以分为5个步骤：

- 用户可以把新建文档提交给任意一个Replica。

- 如果它不是leader，它会把请求转给和自己同Shard的Leader。

- Leader把文档路由给本Shard的每个Replica。

- 如果文档基于路由规则(如取hash值)并不属于当前的Shard，leader会把它转交给对应Shard的Leader。

- 对应Leader会把文档路由给本Shard的每个Replica

## SolrCloud索引的检索

在创建好索引的基础上，SolrCloud检索索引相对就比较简单了：

1、用户的一个查询，可以发送到含有该Collection的任意Solr的Server，Solr内部处理的逻辑会转到一个Replica。

2、此Replica会基于查询索引的方式，启动分布式查询，基于索引的Shard的个数，把查询转为多个子查询，并把每个子查询定位到对应Shard的任意一个Replica。

3、每个子查询返回查询结果。

4、最初的Replica合并子查询，并把最终结果返回给用户。

## SolrCloud中提供NRT近实时搜索：

SolrCloud支持近实时搜索，所谓的近实时搜索即在较短的时间内使得新添加的document可见可查，这主要基于softcommit机制(注意：Lucene是没有softcommit的，只有hardcommit)。上面提到Solr建索引时的数据是在提交时写入磁盘的，这是硬提交，硬提交确保了即便是停电也不会丢失数据；为了提供更实时的检索能力，Solr提供了一种软提交方式。软提交（soft commit）指的是仅把数据提交到内存，index可见，此时没有写入到磁盘索引文件中。在设计中一个通常的做法是：每1-10分钟自动触发硬提交，每秒钟自动触发软提交，当进行softcommit时候，Solr会打开新的Searcher从而使得新的document可见，同时Solr还会进行预热缓存及查询以使得缓存的数据也是可见的，这就必须保证预热缓存以及预热查询的执行时间必须短于commit的频率，否则就会由于打开太多的searcher而造成commit失败。

