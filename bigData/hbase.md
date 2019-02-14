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
