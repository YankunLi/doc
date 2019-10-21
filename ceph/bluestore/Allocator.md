# Allocator

## BitmapAllocator

BitmapAllocator是Allocator的子类.

* Allocator* Allocator::create(CephContext* cct, string type,int64_t size, int64_t block_size)
创建Bluestore的分配器,Bluestore提供了两种类型的分配器:StupidAllocator和BitmapAllocator.该方法会根据参数type创建Allocate实例.默认使用BitmapAllocator.

* void Allocator::release(const PExtentVector& release_vec)
释放release_vec描述的空间.具体实现方法由其子类的release方法实现.
