# 计算机的Cache和Memory访问时Write-back Write-through及wriet allocate的区别

计算机的存储系统采用Register，Cache，Memory和I/O的方式来构成存储系统，无疑是一个性能和经济性的妥协的产物。Cache和Memory机制是计算机硬件的基础内容，这里就不再啰嗦。下面重点说明Write-back，Write-through及write allocate这三种操作的区别。
## CPU 读Cache:
 * Read through, 即直接从内存中读数据;
 * Read allocate, 先把数据读到Cache中，再从Cache中读取数据;

## CPU写Cache:
1. 若hit命中,有两中处理方式:
 * Write-through: write is done synchronously both to the cache and to the backing store。Write-through（直写模式）在数据更新时，把数据同时写入Cache和后端存储。此模式的优点是操作简单；缺点是因为数据修改需要同时写入存储，数据写入速度较慢.
 * Write-back (also called write-behind): initially, writing is done only to the cache. The write to the backing store is postponed until the cache blocks containing the data are about to be modified/replaced by new content。Write-back（回写模式）在数据更新时只写入缓存Cache。只在数据被替换出缓存时，被修改的缓存数据才会被写到后端存储（即先把数据写到Cache中，再通过flush方式写入到内存中）。此模式的优点是数据写入速度快，因为不需要写存储；缺点是一旦更新后的数据未被写入存储时出现系统掉电的情况，数据将无法找回。
2. 若miss,两种处理方式:

[ss](http://test.com)

![st](http://test.com)
