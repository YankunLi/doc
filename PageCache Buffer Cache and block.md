# PageCache Buffer Cahce and Block

## Page

Page是linux描述内存的一个数据结构,一般大小位4kb.该结构定义在/include/linux/mm_types.h头文件中,总字节大小是32字节.该结构记录了相应内存的状态信息,文件系统,内存交换等信息,但是没有明确说明其对应的物理内存偏移位置,即其所表示的内存范围在物理空间的位置,其实这个信息保存在一个数组的下标中,数组mem_map保存了所有的page实例,linux利用该数组的下标来表示page与物理存储的地址映射关系.

```c
/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page, though if it is a pagecache page, rmap structures can tell us
 * who is mapping it.
 */
struct page {
        unsigned long flags;            /* Atomic flags, some possibly
                                         * updated asynchronously */
        atomic_t _count;                /* Usage count, see below. */
        union {
                atomic_t _mapcount;     /* Count of ptes mapped in mms,
                                         * to show when page is mapped
                                         * & limit reverse map searches.
                                         */
                struct {                /* SLUB */
                        u16 inuse;
                        u16 objects;
                };
        };
        union {
            struct {
                unsigned long private;          /* Mapping-private opaque data:
                                                 * usually used for buffer_heads
                                                 * if PagePrivate set; used for
                                                 * swp_entry_t if PageSwapCache;
                                                 * indicates order in the buddy
                                                 * system if PG_buddy is set.
                                                 */
                struct address_space *mapping;  /* If low bit clear, points to
                                                 * inode address_space, or NULL.
                                                 * If page mapped as anonymous
                                                 * memory, low bit is set, and
                                                 * it points to anon_vma object:
                                                 * see PAGE_MAPPING_ANON below.
                                                 */
            };
#if USE_SPLIT_PTLOCKS
            spinlock_t ptl;
#endif
            struct kmem_cache *slab;    /* SLUB: Pointer to slab */
            struct page *first_page;    /* Compound tail pages */
        };
        union {
                pgoff_t index;          /* Our offset within mapping. */
                void *freelist;         /* SLUB: freelist req. slab lock */
        };
        struct list_head lru;           /* Pageout list, eg. active_list
                                         * protected by zone->lru_lock !
                                         */
        /*
         * On machines where all RAM is mapped into kernel address space,
         * we can simply calculate the virtual address. On machines with
         * highmem some memory is mapped into kernel virtual memory
         * dynamically, so we need a place to store that address.
         * Note that this field could be 16 bits on x86 ... ;)
         *
         * Architectures with slow multiplication can define
         * WANT_PAGE_VIRTUAL in asm/page.h
         */
#if defined(WANT_PAGE_VIRTUAL)
        void *virtual;                  /* Kernel virtual address (NULL if
                                           not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
        unsigned long debug_flags;      /* Use atomic bitops on this */
#endif

#ifdef CONFIG_KMEMCHECK
        /*
         * kmemcheck wants to track the status of each byte in a page; this
         * is a pointer to such a status block. NULL if not tracked.
         */
        void *shadow;
#endif
};

```

## Buffer Head

buffer head结构体定义在/include/linux/buffer_head.h头文件中,它表示实际物理内存映射的块设备偏移位置.该结构体保存了内存中page到块设备的映射关系.

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
