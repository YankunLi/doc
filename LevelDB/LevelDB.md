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
