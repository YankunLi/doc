# 计算机的Cache和Memory访问时Write-back Write-through及wriet allocate的区别

计算机的存储系统采用Register，Cache，Memory和I/O的方式来构成存储系统，无疑是一个性能和经济性的妥协的产物。Cache和Memory机制是计算机硬件的基础内容，这里就不再啰嗦。下面重点说明Write-back，Write-through及write allocate这三种操作的区别。
## CPU 读Cache:
 * Read through, 即直接从内存中读数据;
 * Read allocate, 先把数据读到Cache中，再从Cache中读取数据;

## CPU写Cache:

[ss](http://test.com)
![st](http://test.com)
