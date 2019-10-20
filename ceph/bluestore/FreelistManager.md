# FreelistManager

## BitmapFreelistManager

BitmapFreelistManager是FreelistManager的子类,该类主要实现了位图的方式管理磁盘空闲空间.BitmapFreelistManager将磁盘的状态持久化到KV(RocksDB)存储系统中.
BitmapFreelistManager对磁盘空间顺序性编址格式话,将地址空间分三部分,这里称第一部分位段地址,第二部分位块地址和块,即:BitmapFreelistManager中包含若干段,每个段包含固定的block,假设64位表示地址空间,块大小8MB,每个段4096个块,如下位图示意:
 23~1 为块内地址,大小8MB;
 36~24 为每个段内的块地址,可标识4096个块;
 64~37 为段地址,可标识134217728个

BitmapFreelistManager使用一个比特标记block是否被使用或者闲置,只有这两种状态相互转化,所以使用异或(对应方法_xor)运算来改变block的状态,这样就使block的分配和释放方法(release/allocate)基本一样.

* int BitmapFreelistManager::create(uint64_t new_size, uint64_t granularity,KeyValueDB::Transaction txn)

* BitmapFreelistManager::BitmapFreelistManager(CephContext* cct,string meta_prefix,string bitmap_prefix)
该构造方法初始化了meta_prefix/bitmap_prefix/enumerate_bl_pos,并调用了父类的够着方法FreelistManager.

* int BitmapFreelistManager::create(uint64_t new_size, uint64_t granularity,KeyValueDB::Transaction txn)
初始化BitmapFreelistManager,如:磁盘空间总大小 ,划分的每个块的大小, 总的块数, 每个key管理的block数量; 这些信息都保存到KV存储中,防止配置的变动,映像对磁盘的格式变化;(将最后一个key对应的blocks中的超出size的那些blocks置位已分配,以为这些blocks本身不存在,置为已使用防止后期被分配出去.)

* void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length,KeyValueDB::Transaction txn)
将在offset~offset+length区间的block状态反正,并持久化到KV存储中,KB中的key为bitmap_prefix+所在段偏移地址;

* void BitmapFreelistManager::release(uint64_t offset, uint64_t length,KeyValueDB::Transaction txn)
将位于offset~offset+length区间内的block状态设置为0,即标记这些block为空闲状态.

* void BitmapFreelistManager::allocate(uint64_t offset, uint64_t length,KeyValueDB::Transaction txn)
将位于offset~offset+length区间内的block状态设置为1,即标记这些block为已分配状态.

int BitmapFreelistManager::init(KeyValueDB *kvdb)
从KV DB中加载之前持久化的BitmapFreelistManager原信息(mete_prefix+/bytes_per_block/size/blocks/blocks_per_key),来初始化BitmapFreelistManager内存对象,并调用_init_misc()初始化其他BitmapFreelistManager中的内存属性,

* void BitmapFreelistManager::_init_misc()
初始化BitmapFreelistManager中的,属性all_set_bl为1,block_mask/key_mask/bytes_per_key.

* void BitmapFreelistManager::shutdown()
关闭BitmapFreelistManager,实现为空.

* int BitmapFreelistManager::expand(uint64_t new_size, KeyValueDB::Transaction txn)
扩容BitmapFreelistManager所管理的空间,修改之前持久化的BitmapFreelistManager所管理的空间大小size和blocks.

* void BitmapFreelistManager::_verify_range(KeyValueDB *kvdb,uint64_t offset, uint64_t length,int val)
检查offset~offset+length区间内的blocks的状态是否与val相同.不同就会报错.

注:enumerate_reset和enumerate_next是用于迭代BitmapFreelistManager管理的block空间的.enumerate_reset重置由于迭代的内部变量,neumerate_next联系的查找联系的空闲空间.

* void BitmapFreelistManager::enumerate_reset()
重置迭代BitmapFreelistManager管理的blocks空间使用的内部变量:
enumerate_p 用于迭代KV中段key的迭代器.
enumerate_bl 保存每次段key对应的KV中的value.
enumerate_offset enumerate_p所指向的段,在全局磁盘空间中的偏移位置.
enumerate_bl_pos 段内的偏移位置.

* bool BitmapFreelistManager::enumerate_next(KeyValueDB *kvdb, uint64_t *offset, uint64_t *length)
迭代的获取连续空闲的空间,每次获取一个联系的空间段,该空闲区间通过offset,length表示.

* void BitmapFreelistManager::dump(KeyValueDB *kvdb)
打印BitmapFreelistManager中所有空闲的空间区间.

* int get_next_clear_bit(bufferlist& bl, int start)
获取bl表示的比特流中,从start位置开始,首个为bit为0的段内偏移位.

* int get_next_set_bit(bufferlist& bl, int start)
获取bl表示的比特流中,从start位置开始,首个为bit为1的段内偏移位.

* uint64_t _get_offset(uint64_t key_off, int bit)
获取段偏移为key_off,段内block偏移bit的,在全局空间中的偏移位置.

* void make_offset_key(uint64_t offset, std::string *key)
将uint64_t offset转换成string key.

## XorMergeOperator

XorMergeOperator继承了KeyValueDB::MergeOperator, 主要实现merge方法函数,实现具体的merge/merge_nonexistent算法(KV存储支持特性操作merge,并且具体的merge算法可以自定义),merge的自定义实现是将给的参数,按位做异或运算.

* void merge(const char *ldata, size_t llen,const char *rdata, size_t rlen,std::string *new_value)
将ldata与rdata两个字符串按位做异或运算,结果保存到new_value中返回.

* void merge_nonexistent(const char *rdata, size_t rlen, std::string *new_value)
对不存在原始内容做merge操作,即直接返回新添加的内容即可.

* const char *name()
返回"bitwise_xor".用于rocksdb构造时使用的merge操作名字.
