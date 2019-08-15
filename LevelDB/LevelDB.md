# LevelDB 纪要：

## LevelDB 整体架构
1. leveldb存储到磁盘上的数据是根据key值有序存储的；
2. 用户可以自定义key值的比较函数，按照用户定义的比较规则有序存储
3. 支持读写删除、和多条操作的原则批量操作；
4. leveldb支持快照功能；
5. leveldb支持压缩功能；
6. leveldb在写入一条记录时：主要包括两步：1.先写入日志文件（WAL）；2.更新内存中memtable表，即一次操作包括一次顺序磁盘写入，和内存写入所以速度很快；
7. 根据6，系统崩溃时可以通过日志，恢复内存中没有dump到磁盘的数据；
8. 内存中的memtable的量达到一定上限，就会将其转换成immutable memtable，并生成新的log文件和memtable，继续接收新的写入请求，immutable memtable会被compaction操作写入到磁盘生成新的sstable文件；
9. 所有sstable文件是采用层级结构形式组织的，由level0~leveln；
10. sstable中的记录是根据key值有序存储的，并且level0中，两个文件有可能存在相同的key中重叠（为什么），其他层不会出现重叠现象（为什么）
11. manifest文件管理所有的sstable文件信息，维护不同sstable文件所在层级，及该文件中key的最大最小值信息；
12. current文件是用来指向manifest文件的，其内容就是manifest文件名，以为leveldb中的sstable文件是不断变化的，manifest文件也会跟着变化，所有current文件就是告诉我们那个manifest文件是当前可信的；

## LevelDB Log
1. LevelDB 的log文件被等大小的逻辑划分成若干块,每块的大小位32KB,每次读取块大小的数据;
2. LevelDB中的日子记录都有一个统一的格式,其中包括记录头(checksum,记录长度,类型), 记录体(数据):  
   **checksum|记录长度|类型|数据**
   1. **checksum** 时类型和数据的的校验码,用于检查数据在存取过程中是否发生变化;
   2. **记录长度** 记录数据的大小;
   3. **类型** 记录该记录的逻辑结构与log的逻辑分块之间的关系, 其取值有四种:**FULL/FIRST/MIDDLE/LAST**:
      1. **FULL** 表示该记录在log的一个block中;
      2. **FIRST** 表示在当前block中只存放了记录的开始一部分数据,下一个block中还有该记录的数据;
      3. **MIDDLE** 表示在该block中只保存了一条记录的中间部分,其前后部分数据都保存在相邻的block中;
      4. **LAST** 表示在该block中保存了一条记录的剩余尾巴数据;
       **如果一条记录只保存在一个block中,则类型位FULL,否则会有FIRST/MIDDLE/LAST类型, MIDDLE只有在记录横跨的block数大于等于3时才有MIDDLE类型**
      > 注: LevelDB读取时,每次读取一个block大小,再根据其中记录的类型,来决定是否继续读取,来补全逻辑记录;

## SSTable文件:
### SSTable文件的内部数据组织结构  
#### SSTable文件的逻辑划分:
1. sstable文件被逻辑划分成固定等大小的block;
2. 每个block中,包含3部分:**数据|Type|CRC**
   1. 数据时存储的主体数据(KV值)
   2. Type表示是否采用压缩算法(snappy压缩算法或者无压缩算法)
   3. CRC用于校验数据,在中间处理过程中是否发生变化;
#### SSTable中数据内容的逻辑布局:
数据内容的逻辑布局都是在文件逻辑划分的基础上建立的.
1. SSTable中的内容大体可以分两部分:
   1. 数据区, 位于文件的前部分;
   2. 数据管理区,位于文件的后部分;
2. 数据区保存了KV的实际数据;
3. 数据管理区提供一些索引指针等管理数据,目的是更快捷便利的查找相应的记录;
4. 管理区的数据由根据作用不同,划分成4种不同类型:
   1. 元数据
   2. 元数据的索引
   3. 数据索引
   4. 文件尾部块
   > 注: LevelDB1.2中尚未使用元数据,和元数据索引;

1. 数据索引和Footer结构,如下图:

![data index and footer](https://github.com/YankunLi/doc/blob/master/LevelDB/data_index_and_Footer.jpg#pic_center "数据索引和SSTable尾部结构")

