# PageCache Buffer Cahce and Block

## Page

Page是linux描述内存的一个数据结构,一般大小位4kb.该结构定义在/include/linux/mm_types.h头文件中,总字节大小是32字节.该结构记录了相应内存的状态信息,文件系统,内存交换等信息,但是没有明确说明其对应的物理内存偏移位置,即其所表示的内存范围在物理空间的位置,其实这个信息保存在一个数组的下标中,数组mem_map保存了所有的page实例,linux利用该数组的下标来表示page与物理存储的地址映射关系.

```

```