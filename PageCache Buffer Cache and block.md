# PageCache Buffer Cahce and Block

## Page

Page是linux描述内存的一个数据结构,一般大小位4kb.该结构定义在/include/linux/mm_types.h头文件中,总字节大小是32字节.该结构记录了相应内存的状态信息,文件系统,内存交换等信息,但是没有明确说明其对应的物理内存偏移位置,即其所表示的内存范围在物理空间的位置,其实这个信息保存在一个数组的下标中,数组mem_map保存了所有的page实例,linux利用该数组的下标来表示page与物理存储的地址映射关系.

```

```

## Buffer Head

buffer head结构体定义在/include/linux/buffer_head.h头文件中,它表示实际物理内存映射的块设备偏移位置.该结构体保存了内存中一块空间与块设备的映射关系.

```c
/*
 * Historically, a buffer_head was used to map a single block
 * within a page, and of course as the unit of I/O through the
 * filesystem and block layers.  Nowadays the basic I/O unit
 * is the bio, and buffer_heads are used for extracting block
 * mappings (via a get_block_t call), for tracking state within
 * a page (via a page_mapping) and for wrapping bio submission
 * for backward compatibility reasons (e.g. submit_bh).
 */
struct buffer_head {
        unsigned long b_state;          /* buffer state bitmap (see above) */
        struct buffer_head *b_this_page;/* circular list of page's buffers */
        struct page *b_page;            /* the page this bh is mapped to */

        sector_t b_blocknr;             /* start block number */
        size_t b_size;                  /* size of mapping */
        char *b_data;                   /* pointer to data within the page */

        struct block_device *b_bdev;
        bh_end_io_t *b_end_io;          /* I/O completion */
        void *b_private;                /* reserved for b_end_io */
        struct list_head b_assoc_buffers; /* associated with another mapping */
        struct address_space *b_assoc_map;      /* mapping this buffer is
                                                   associated with */
        atomic_t b_count;               /* users using this buffer_head */
};

```
