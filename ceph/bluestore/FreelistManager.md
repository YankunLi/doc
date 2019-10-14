# FreelistManager

## BitmapFreelistManager

BitmapFreelistManager是FreelistManager的子类,该类主要实现了位图的方式管理磁盘空闲空间.BitmapFreelistManager将磁盘的状态持久化到KV(RocksDB)存储系统中.

* int BitmapFreelistManager::create(uint64_t new_size, uint64_t granularity,KeyValueDB::Transaction txn)

* BitmapFreelistManager::BitmapFreelistManager(CephContext* cct,string meta_prefix,string bitmap_prefix)
该构造方法初始化了meta_prefix/bitmap_prefix/enumerate_bl_pos,并调用了父类的够着方法FreelistManager.
