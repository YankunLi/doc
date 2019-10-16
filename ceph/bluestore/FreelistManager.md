# FreelistManager

## BitmapFreelistManager

BitmapFreelistManager是FreelistManager的子类,该类主要实现了位图的方式管理磁盘空闲空间.BitmapFreelistManager将磁盘的状态持久化到KV(RocksDB)存储系统中.
BitmapFreelistManager对磁盘空间顺序性编址格式话,将地址空间分三部分,这里称第一部分位段地址,第二部分位块地址和块,即:BitmapFreelistManager中包含若干段,每个段包含固定的block,假设64位表示地址空间,块大小8MB,每个段4096个块,如下位图示意:
 23~1 为块内地址,大小8MB;
 36~24 为每个段内的块地址,可标识4096个块;
 64~37 为段地址,可标识134217728个

* int BitmapFreelistManager::create(uint64_t new_size, uint64_t granularity,KeyValueDB::Transaction txn)

* BitmapFreelistManager::BitmapFreelistManager(CephContext* cct,string meta_prefix,string bitmap_prefix)
该构造方法初始化了meta_prefix/bitmap_prefix/enumerate_bl_pos,并调用了父类的够着方法FreelistManager.

* int BitmapFreelistManager::create(uint64_t new_size, uint64_t granularity,KeyValueDB::Transaction txn)
初始化BitmapFreelistManager,如:磁盘空间总大小 ,划分的每个块的大小, 总的块数, 每个key管理的block数量; 这些信息都保存到KV存储中,防止配置的变动,映像对磁盘的格式变化;

* void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length,KeyValueDB::Transaction txn)
将在offset~offset+length区间的block状态反正,并持久化到KV存储中,KB中的key为bitmap_prefix+所在段偏移地址;
