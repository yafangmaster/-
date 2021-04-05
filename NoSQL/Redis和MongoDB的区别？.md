## Redis和MongoDB的区别？
### 1.内存管理
Redis数据全部存入内存，定期写入磁盘，当内存不足时，可以选择指定的LRU算法删除数据；

MongoDB数据会优先存入内存，当内存不足时，只将热点数据写入内存，其他数据存在磁盘；
### 2.数据结构
Redis支持数据结构丰富，包含：hash、list、set等；

MongoDB数据结构比较单一，但是支持丰富的数据表达、索引，最类似关系数据库，支持的查询语言比较丰富；
### 3.数据量和性能
当物理内存够用的时候，性能：Redis > MongoDB > MySQL；

数据量：MySQL > MongoDB > Redis

注意：MongoDB可以存储文件，适合存放大量的小文件，内置GirdFS的分布式文件系统；
### 4.可靠性
Redis依赖快照进行持久化；AOF增强可靠性，增强可靠性的同事，影响访问性能；

MongoDB从1.8版本后，采用binlog方式支持持久化，增强可靠性；

参考链接：
https://zhuanlan.zhihu.com/p/86777551
