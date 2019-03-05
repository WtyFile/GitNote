#### HBase的优点：
列可以动态增加，并且列为空就不存储数据，节省存储空间。
Hbase自动切分数据，使得数据存储自动具有水平scalability。
Hbase可以提供高并发读写操作的支持。
 ![image.png](0)
#### Client
包含访问HBase的接口并维护cache来加快对HBase的访问
#### Zookeeper
保证任何时候，集群中只有一个Master
存储所有Region的寻址入口
实时监控RegionServer的上线和下线信息，并实时通知Master
储存Hbase的schema和table的元数据
#### Master
为RegionServer分配region，发现失效的RegionServer并重新分配其上的Region
负责RegionServer的负载均衡（避免所有请求到一个RegionServer中去）
管理用户对table的增删改操作（table存在于region里面）
#### RegionServer
RegionServer维护Region，处理对Region的io请求
RegionServer负责切分在运行过程中变得过大的Region
#### Region（一个表由多个或单个region组成）
HBase自动把表水平划分成多个区域（region），每个region会保存一个表里面某段连续（关系到rowkey的设计，rowkey的设计关系到查询效率的高低）的数据
每个表一开始只有一个region，随着数据不断插入表，region会不断增大，当增大到一个阀值后，region就会等分为两个新的region（裂变）
当table中的行不断增多，就会有越来越多的region，这样一张完整的表就被保存在多个Region甚至多个RegionServer上
#### MemStore与storefile(StoreFile对应HFile，StoreFile以HFile格式保存在HDFS上)（一个store对应一个列族CF）
一个region由多个store组成
store包括位于内存中的memstore和位于磁盘的storefile，写操作先写入memstore，当memstore中的数据量达到某个阀值，hregionserver会启动flashcache进程写入storefile（可以手动启动flashcache进程写入storefile），每次写入形成单独的一个storefile
当storefile文件数量增长到一个阀值后，系统会进行合并（minor（少个文件合并，不会对系统进程影响太大，因此设置自动开启），major（多个文件合并，文件数量过多时，合并会影响系统进程，导致其他进程阻塞，因此设置定时开启，读写数据少的时候开启）），在合并过程中会进行版本合并和删除工作（major）（在hbase表中设置版本数，如果为一，则表示只能存一条版本最新的，老版本并不会直接删除，而是会加上失效的标签，再进行major的时候，才会合并和删除），形成更大的storefile
当一个region中所有storefile的大小和数量超过一定阀值后，会把当前的region分割为两个，并由hmaster分配到相应的regionserver服务器，实现负载均衡
客户端检索数据，先找memstore(主要作用是写数据，但是也可以分担读取压力)，找不到再找storefile
rowkey设计原则和方法
rowkey设计首先应当遵循三大原则：

rowkey长度原则
rowkey是一个二进制码流，可以为任意字符串，最大长度为64kb，实际应用中一般为10-100bytes，它以byte[]形式保存，一般设定成定长。

一般越短越好，不要超过16个字节，注意原因如下：

1、目前操作系统都是64位系统，内存8字节对齐，控制在16字节，8字节的整数倍利用了操作系统的最佳特性。

2、hbase将部分数据加载到内存当中，如果rowkey过长，内存的有效利用率就会下降。

rowkey散列原则
如果rowkey按照时间戳的方式递增，不要将时间放在二进制码的前面，建议将rowkey的高位字节采用散列字段处理，由程序随即生成。低位放时间字段，这样将提高数据均衡分布，各个regionServer负载均衡的几率。

如果不进行散列处理，首字段直接使用时间信息，所有该时段的数据都将集中到一个regionServer当中，这样当检索数据时，负载会集中到个别regionServer上，造成热点问题，会降低查询效率。

rowkey唯一原则
必须在设计上保证其唯一性，rowkey是按照字典顺序排序存储的，因此，设计rowkey的时候，要充分利用这个排序的特点，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。但是这里的量不能太大，如果太大需要拆分到多个节点上去。
#### 热点问题
###### 热点的危害：
大量访问会使热点region所在的单个主机负载过大，引起性能下降甚至region不可用。

###### 热点产生原因：
有大量连续编号的row key  ==>  大量row key相近的记录集中在个别region

 ==>  client检索记录时,对个别region访问过多  ==>  此region所在的主机过载  ==>  热点
###### 避免方法（随机散列与预分区）和优缺点
加盐
       这里所说的加盐不是密码学中的加盐，而是在rowkey的前面增加随机数，具体就是给rowkey分配一个随机前缀以使得它和之前的rowkey的开头不同。给多少个前缀？这个数量应该和我们想要分散数据到不同的region的数量一致（类似hive里面的分桶）。加盐之后的rowkey就会根据随机生成的前缀分散到各个region上，以避免热点。
哈希
       哈希会使同一行永远用一个前缀加盐。哈希也可以使负载分散到整个集群，但是读却是可以预测的。使用确定的哈希可以让客户端重构完整的rowkey，可以使用get操作准确获取某一个行数据。
反转
       第三种防止热点的方法是反转固定长度或者数字格式的rowkey。这样可以使得rowkey中经常改变的部分（最没有意义的部分）放在前面。这样可以有效的随机rowkey，但是牺牲了rowkey的有序性。

反转rowkey的例子：以手机号为rowkey，可以将手机号反转后的字符串作为rowkey，从而避免诸如139、158之类的固定号码开头导致的热点问题。
时间戳反转
       一个常见的数据处理问题是快速获取数据的最近版本，使用反转的时间戳作为rowkey的一部分对这个问题十分有用，可以用Long.Max_Value - timestamp追加到key的末尾，例如[key][reverse_timestamp] ,[key] 的最新值可以通过scan [key]获得[key]的第一条记录，因为HBase中rowkey是有序的，第一条记录是最后录入的数据。
其他办法
	列族名的长度尽可能小，最好是只有一个字符。冗长的属性名虽然可读性好，但是更短的属性名存储在HBase中会更好。也可以在建表时预估数据规模，预留region数量，例如create 'myspace:mytable’, SPLITS => [01,02,03,,...99]