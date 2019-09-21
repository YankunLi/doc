# Kernel fuse

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
        struct file *file;
        ssize_t ret = -EBADF;
        int fput_needed;

        file = fget_light(fd, &fput_needed); //
        if (file) {
                loff_t pos = file_pos_read(file);
                ret = vfs_read(file, buf, count, &pos);
                file_pos_write(file, pos);
                fput_light(file, fput_needed);
        }

        return ret;
}

```

通过指定的文件描述付,获取对应的vfs中文件对象;
获取文件对象中的当前偏移位置;调用vfs_read方法发起读请求(指定文件对象,存储请求数据的空间,读取字节大小,即读取的起始偏移位置);
更新文件对象的偏移位置,无论操作是否成功都会更新文件对象的偏移位置;

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
        ssize_t ret;

        if (!(file->f_mode & FMODE_READ))
                return -EBADF;
        if (!file->f_op || (!file->f_op->read && !file->f_op->aio_read))
                return -EINVAL;
        if (unlikely(!access_ok(VERIFY_WRITE, buf, count)))
                return -EFAULT;

        ret = rw_verify_area(READ, file, pos, count);
        if (ret >= 0) {
                count = ret;
                if (file->f_op->read)
                        ret = file->f_op->read(file, buf, count, pos);
                else
                        ret = do_sync_read(file, buf, count, pos);
                if (ret > 0) {
                        fsnotify_access(file->f_path.dentry);
                        add_rchar(current, ret);
                }
                inc_syscr(current);
        }

        return ret;
}
```

vfs_read对输入参数做合法性检查,文件对象是否可读,文件操作函数对象是否初始化,用于保存读取的数据的buffer空间是否可写;
rw_verify_area对操作文件对象是做锁检查和操作合法性检查;
如果文件对象的操作对象f_op的read被初始化就调用该方法,继续执行读操作,否则调用do_sync_read函数执行读操作;
如果成功就发起操作的文件对应的inode变动事件,累加当前进程读取的数据总量;
累加进程系统调用次数;

```c
static const struct file_operations fuse_file_operations = {
        .llseek         = fuse_file_llseek,
        .read           = do_sync_read,
        .aio_read       = fuse_file_aio_read,
        .write          = do_sync_write,
        .aio_write      = fuse_file_aio_write,
        .mmap           = fuse_file_mmap,
        .open           = fuse_open,
        .flush          = fuse_flush,
        .release        = fuse_release,
        .fsync          = fuse_fsync,
        .lock           = fuse_file_lock,
        .flock          = fuse_file_flock,
        .splice_read    = generic_file_splice_read,
        .unlocked_ioctl = fuse_file_ioctl,
        .compat_ioctl   = fuse_file_compat_ioctl,
        .poll           = fuse_file_poll,
};
```

fuse文件系统类型,文件对象的文件操作对象的初始化;read属性和write属性分别被初始化do_sync_read和do_sync_write;

```c
ssize_t do_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
        struct iovec iov = { .iov_base = buf, .iov_len = len };
        struct kiocb kiocb;
        ssize_t ret;

        init_sync_kiocb(&kiocb, filp);
        kiocb.ki_pos = *ppos;
        kiocb.ki_left = len;

        for (;;) {
                ret = filp->f_op->aio_read(&kiocb, &iov, 1, kiocb.ki_pos);
                if (ret != -EIOCBRETRY)
                        break;
                wait_on_retry_sync_kiocb(&kiocb);
        }

        if (-EIOCBQUEUED == ret)
                ret = wait_on_sync_kiocb(&kiocb);
        *ppos = kiocb.ki_pos;
        return ret;
}
```

初始化iovec对象,主要是存储空间和操作的范围(大小);
初始化kiocb,该对象用于跟踪IO的完成情况;(注:通常块设备和网络设备都是异步IO执行,字符设备同步执行;)
调用文件对象的操作对象aio_read函数对象继续读取数据,aio_read对应fuse的fuse_file_aio_read函数;
文件读取操作一次操作可能不能完成读请求的数据,需要多次操作,直到不需要在retry为止;
读操作在未完成时进程主动调用进程调度函数让出cpu,进程设置TASK_UNINTERRUPTIBLE状态,直到读取成功进程设置成TASK_RUNNING状态;
do_sync_read将读请求的参数转换成iovvec,并创建辅助变量kiocb实现异步执行,然后向具体的文件系统发起请求;

如下: wait_on_retry_sync_kiocb函数;

```c
static void wait_on_retry_sync_kiocb(struct kiocb *iocb)
{
        set_current_state(TASK_UNINTERRUPTIBLE);
        if (!kiocbIsKicked(iocb))
                schedule();
        else
                kiocbClearKicked(iocb);
        __set_current_state(TASK_RUNNING);
}
```

文件对象的文件操作对象的aio_read属性被初始化fuse_file_aio_read

```c
static ssize_t fuse_file_aio_read(struct kiocb *iocb, const struct iovec *iov,
                                  unsigned long nr_segs, loff_t pos)
{
        struct inode *inode = iocb->ki_filp->f_mapping->host;

        if (pos + iov_length(iov, nr_segs) > i_size_read(inode)) {
                int err;
                /*
                 * If trying to read past EOF, make sure the i_size
                 * attribute is up-to-date.
                 */
                err = fuse_update_attributes(inode, NULL, iocb->ki_filp, NULL);
                if (err)
                        return err;
        }

        return generic_file_aio_read(iocb, iov, nr_segs, pos);
}
```

如果要读取的范围超出了文件的大小,就更新一些内存中inode的size(有可能文件大小已经发生改变但是内存中没有更新);
调用generic_file_aio_read继续执行读取操作;

```c
/**
 * generic_file_aio_read - generic filesystem read routine
 * @iocb:       kernel I/O control block
 * @iov:        io vector request
 * @nr_segs:    number of segments in the iovec
 * @pos:        current file position
 *
 * This is the "read()" routine for all filesystems
 * that can use the page cache directly.
 */
ssize_t
generic_file_aio_read(struct kiocb *iocb, const struct iovec *iov,
                unsigned long nr_segs, loff_t pos)
{
        struct file *filp = iocb->ki_filp;
        ssize_t retval;
        unsigned long seg;
        size_t count;
        loff_t *ppos = &iocb->ki_pos;

        count = 0;
//在写buffer之前对该空间做必要的可写性检查
        retval = generic_segment_checks(iov, &nr_segs, &count, VERIFY_WRITE);
        if (retval)
                return retval;

        /* coalesce the iovecs and go direct-to-BIO for O_DIRECT */
//如果打开文件时,设置了O_DIRECT flag会走如下流程:将读取范围中的脏页写会,绕过pagecache直接发起IO请求.
        if (filp->f_flags & O_DIRECT) {
                loff_t size;
                struct address_space *mapping;
                struct inode *inode;

                mapping = filp->f_mapping;
                inode = mapping->host;
                if (!count)
                        goto out; /* skip atime */
                size = i_size_read(inode);
                if (pos < size) {
                    //将读操作范围中的脏页写会,并等待其完成
                        retval = filemap_write_and_wait_range(mapping, pos,
                                        pos + iov_length(iov, nr_segs) - 1);
                        if (!retval) {
//绕过pagecache层,直接读取数据
                                retval = mapping->a_ops->direct_IO(READ, iocb,
                                                        iov, pos, nr_segs);
                        }
                        if (retval > 0)
                                *ppos = pos + retval;
                        if (retval) {
                                file_accessed(filp);
                                goto out;
                        }
                }
        }
//如果文件文件没有被这种O_DIRECT flag则走如下流程:经过pagecache层.
        for (seg = 0; seg < nr_segs; seg++) {
                read_descriptor_t desc;

                desc.written = 0;
                desc.arg.buf = iov[seg].iov_base;
                desc.count = iov[seg].iov_len;
                if (desc.count == 0)
                        continue;
                desc.error = 0;
                do_generic_file_read(filp, ppos, &desc, file_read_actor);
                retval += desc.written;
                if (desc.error) {
                        retval = retval ?: desc.error;
                        break;
                }
                if (desc.count > 0)
                        break;
        }
out:
        return retval;
}
````

generic_file_aio_read函数执行文件内容读取工作,根据文件描述符创建时如果带O_DIRECT标志,则将内存中的脏也写回磁盘,再通过address_space的操作对象中的direct_IO函数指针指向的函数执行读取操作(即直接从文件中读取数据),如果不带O_DIRECT标志,则使用do_generic_file_read函数继续读取数据.

generic_file_aio_read该函数是所有文件系统read操作的通用函数,通过该函数可以使用pagecache特性功能;

```c
/**
 * filemap_write_and_wait_range - write out & wait on a file range
 * @mapping:    the address_space for the pages
 * @lstart:     offset in bytes where the range starts
 * @lend:       offset in bytes where the range ends (inclusive)
 *
 * Write out and wait upon file offsets lstart->lend, inclusive.
 *
 * Note that `lend' is inclusive (describes the last byte to be written) so
 * that this function can be used to write to the very end-of-file (end = -1).
 */
int filemap_write_and_wait_range(struct address_space *mapping,
                                 loff_t lstart, loff_t lend)
{
        int err = 0;

        if (mapping->nrpages) {
                err = __filemap_fdatawrite_range(mapping, lstart, lend,
                                                 WB_SYNC_ALL);
                /* See comment of filemap_write_and_wait() */
                if (err != -EIO) {
                        int err2 = wait_on_page_writeback_range(mapping,
                                                lstart >> PAGE_CACHE_SHIFT,
                                                lend >> PAGE_CACHE_SHIFT);
                        if (!err)
                                err = err2;
                }
        }
        return err;
}
```

将lstart到lend范围的脏页,写回磁盘,直到写入成功才返回.

```c
/**
 * do_generic_file_read - generic file read routine
 * @filp:       the file to read
 * @ppos:       current file position
 * @desc:       read_descriptor
 * @actor:      read method
 *
 * This is a generic file read routine, and uses the
 * mapping->a_ops->readpage() function for the actual low-level stuff.
 *
 * This is really ugly. But the goto's actually try to clarify some
 * of the logic when it comes to error handling etc.
 */
static void do_generic_file_read(struct file *filp, loff_t *ppos,
                read_descriptor_t *desc, read_actor_t actor)
{
        struct address_space *mapping = filp->f_mapping;
        struct inode *inode = mapping->host;
        struct file_ra_state *ra = &filp->f_ra;
        pgoff_t index;
        pgoff_t last_index;
        pgoff_t prev_index;
        unsigned long offset;      /* offset into pagecache page */
        unsigned int prev_offset;
        int error;

        index = *ppos >> PAGE_CACHE_SHIFT;
        prev_index = ra->prev_pos >> PAGE_CACHE_SHIFT;
        prev_offset = ra->prev_pos & (PAGE_CACHE_SIZE-1);
        last_index = (*ppos + desc->count + PAGE_CACHE_SIZE-1) >> PAGE_CACHE_SHIFT;
        offset = *ppos & ~PAGE_CACHE_MASK;

        for (;;) {
                struct page *page;
                pgoff_t end_index;
                loff_t isize;
                unsigned long nr, ret;

                cond_resched();
find_page:      
//在文件的radix树中查找要读取的page
                page = find_get_page(mapping, index);
                if (!page) {
//page为null,说明要读取的页不在radix树中(不在pagecache中),这时进行同步预读操作, 然后再从pagecache中读取页,此时一般都可以命中;
                        page_cache_sync_readahead(mapping,
                                        ra, filp,
                                        index, last_index - index);
                        page = find_get_page(mapping, index);
                        if (unlikely(page == NULL))
                                goto no_cached_page;
                }

                if (PageReadahead(page)) {
//从radix树(pagecache)中命中页,且是预读页,则再做异步预读操作,这里是对后面继续读取的猜测,做的加速优化;
                        page_cache_async_readahead(mapping,
                                        ra, filp, page,
                                        index, last_index - index);
                }

                if (!PageUptodate(page)) {
//在radix中找到了要读取的页,但是该页中没有填充磁盘中的数据,即只分配了空间,但是没有数据,所以需要跳到读取数据逻辑段读取数据;
                        if (inode->i_blkbits == PAGE_CACHE_SHIFT ||
                                        !mapping->a_ops->is_partially_uptodate)
                                goto page_not_up_to_date;
                        if (!trylock_page(page))
                                goto page_not_up_to_date;
                        if (!mapping->a_ops->is_partially_uptodate(page,
                                                                desc, offset))
                                goto page_not_up_to_date_locked;
                        unlock_page(page);
                }
page_ok:
//page_ok段的代码是表示读取到页的内容了,将该页的内容copy到用户空间;,其主要逻辑是参数合法性检查,数据拷贝;
                /*
                 * i_size must be checked after we know the page is Uptodate.
                 *
                 * Checking i_size after the check allows us to calculate
                 * the correct value for "nr", which means the zero-filled
                 * part of the page is not copied back to userspace (unless
                 * another truncate extends the file - this is desired though).
                 */
//合法性检查,是不是长度为0,或者超出文件范围;
                isize = i_size_read(inode);
                end_index = (isize - 1) >> PAGE_CACHE_SHIFT;
                if (unlikely(!isize || index > end_index)) {
                        page_cache_release(page);
                        goto out;
                }
//合法性检查
                /* nr is the maximum number of bytes to copy from this page */
                nr = PAGE_CACHE_SIZE;
                if (index == end_index) {
                        nr = ((isize - 1) & ~PAGE_CACHE_MASK) + 1;
                        if (nr <= offset) {
                                page_cache_release(page);
                                goto out;
                        }
                }
                nr = nr - offset;

                /* If users can be writing to this page using arbitrary
                 * virtual addresses, take care about potential aliasing
                 * before reading the page on the kernel side.
                 */
                if (mapping_writably_mapped(mapping))
                        flush_dcache_page(page);

                /*
                 * When a sequential read accesses a page several times,
                 * only mark it as accessed the first time.
                 */
                if (prev_index != index || offset != prev_offset)
                        mark_page_accessed(page);
                prev_index = index;

                /*
                 * Ok, we have the page, and it's up-to-date, so
                 * now we can copy it to user space...
                 *
                 * The actor routine returns how many bytes were actually used..
                 * NOTE! This may not be the same as how much of a user buffer
                 * we filled up (we may be padding etc), so we can only update
                 * "pos" here (the actor routine has to update the user buffer
                 * pointers and the remaining count).
                 */
//将读取到的页中数据,由内核空间拷贝到用户空间;
                ret = actor(desc, page, offset, nr);
                offset += ret;
                index += offset >> PAGE_CACHE_SHIFT;
                offset &= ~PAGE_CACHE_MASK;
                prev_offset = offset;

                page_cache_release(page);
                if (ret == nr && desc->count)
                        continue;
                goto out;

page_not_up_to_date:
                /* Get exclusive access to the page ... */
//读取第一个page之前要先对该页加锁;
                error = lock_page_killable(page);
                if (unlikely(error))
                        goto readpage_error;

page_not_up_to_date_locked:
                /* Did it get truncated before we got the lock? */
//获取page lock之后发现该page的映射被取消了,可能是在获取锁的过程中被释放了,所以需要从新获取page;
                if (!page->mapping) {
                        unlock_page(page);
                        page_cache_release(page);
                        continue;
                }

                /* Did somebody else fill it already? */
//获取page lock之后,发现page中的数据已经填充,就不需要再读取文件了;
                if (PageUptodate(page)) {
                        unlock_page(page);
                        goto page_ok;
                }

readpage:
                /* Start the actual read. The read will unlock the page. */
//开始实际文件数据读取;
                error = mapping->a_ops->readpage(filp, page);

                if (unlikely(error)) {
                        if (error == AOP_TRUNCATED_PAGE) {
                                page_cache_release(page);
                                goto find_page;
                        }
                        goto readpage_error;
                }

                if (!PageUptodate(page)) {
//lock page等待数据返回,可能会休眠;
                        error = lock_page_killable(page);
                        if (unlikely(error))
                                goto readpage_error;
                        if (!PageUptodate(page)) {
                                if (page->mapping == NULL) {
                                        /*
                                         * invalidate_inode_pages got it
                                         */
                                        unlock_page(page);
                                        page_cache_release(page);
                                        goto find_page;
                                }
                                unlock_page(page);
                                shrink_readahead_size_eio(filp, ra);
                                error = -EIO;
                                goto readpage_error;
                        }
                        unlock_page(page);
                }
//数据读取完成;
                goto page_ok;

readpage_error:
                /* UHHUH! A synchronous read error occurred. Report it */
                desc->error = error;
                page_cache_release(page);
                goto out;

no_cached_page:
                /*
                 * Ok, it wasn't cached, so we need to create a new
                 * page..
                 */
//pagecache, 文件的radix树中,没有对应的page页,需要分配页,然后再读取数据,填充该page;
                page = page_cache_alloc_cold(mapping);
                if (!page) {
                        desc->error = -ENOMEM;
                        goto out;
                }
                error = add_to_page_cache_lru(page, mapping,
                                                index, GFP_KERNEL);
                if (error) {
                        page_cache_release(page);
                        if (error == -EEXIST)
                                goto find_page;
                        desc->error = error;
                        goto out;
                }
                goto readpage;
        }

out:
        ra->prev_pos = prev_index;
        ra->prev_pos <<= PAGE_CACHE_SHIFT;
        ra->prev_pos |= prev_offset;

        *ppos = ((loff_t)index << PAGE_CACHE_SHIFT) + offset;
        file_accessed(filp);

```

do_generic_file_read从pagecache读取要读取数据所在的页:
    如果该页已经存在,但是没有数据,则获取该页,读取文件数据填充该页,并拷贝到用户态(调用者提供的用户空间缓存);
    如果该页存在且有数据,这直接拷贝到用户态(调用者提供的用户空间缓存);
    如果该页不在pagecache中,则申请page页,加入文件的radix树中,然后读取文件内容,填充该页,并将该页数据拷贝的用户态(调用者提供的用户空间缓存)

```c
/**
 * find_get_page - find and get a page reference
 * @mapping: the address_space to search
 * @offset: the page index
 *
 * Is there a pagecache struct page at the given (mapping, offset) tuple?
 * If yes, increment its refcount and return it; if no, return NULL.
 */
struct page *find_get_page(struct address_space *mapping, pgoff_t offset)
{
        void **pagep;
        struct page *page;

        rcu_read_lock();
repeat:
        page = NULL;
        pagep = radix_tree_lookup_slot(&mapping->page_tree, offset);
        if (pagep) {
                page = radix_tree_deref_slot(pagep);
                if (unlikely(!page || page == RADIX_TREE_RETRY))
                        goto repeat;

                if (!page_cache_get_speculative(page))
                        goto repeat;

                /*
                 * Has the page moved?
                 * This is part of the lockless pagecache protocol. See
                 * include/linux/pagemap.h for details.
                 */
                if (unlikely(page != *pagep)) {
                        page_cache_release(page);
                        goto repeat;
                }
        }
        rcu_read_unlock();

        return page;

```

find_get_page从address_space中查找返回指定索引也的引用

```c
/**
 * page_cache_sync_readahead - generic file readahead
 * @mapping: address_space which holds the pagecache and I/O vectors
 * @ra: file_ra_state which holds the readahead state
 * @filp: passed on to ->readpage() and ->readpages()
 * @offset: start offset into @mapping, in pagecache page-sized units
 * @req_size: hint: total size of the read which the caller is performing in
 *            pagecache pages
 *
 * page_cache_sync_readahead() should be called when a cache miss happened:
 * it will submit the read.  The readahead logic may decide to piggyback more
 * pages onto the read request if access patterns suggest it will improve
 * performance.
 */
void page_cache_sync_readahead(struct address_space *mapping,
                               struct file_ra_state *ra, struct file *filp,
                               pgoff_t offset, unsigned long req_size)
{
        /* no read-ahead */
        if (!ra->ra_pages)
                return;

        /* do read-ahead */
        ondemand_readahead(mapping, ra, filp, false, offset, req_size);
}
```

当要读取的页在pagecache中,会执行页同步预读,page_cache_sync_readahead通用的文件同步预读实现,offset 表示要读取数据所在的起始页, req_size涉及的页数,其具体实现由ondemand_readahead完成.

```c
/**
 * page_cache_async_readahead - file readahead for marked pages
 * @mapping: address_space which holds the pagecache and I/O vectors
 * @ra: file_ra_state which holds the readahead state
 * @filp: passed on to ->readpage() and ->readpages()
 * @page: the page at @offset which has the PG_readahead flag set
 * @offset: start offset into @mapping, in pagecache page-sized units
 * @req_size: hint: total size of the read which the caller is performing in
 *            pagecache pages
 *
 * page_cache_async_ondemand() should be called when a page is used which
 * has the PG_readahead flag; this is a marker to suggest that the application
 * has used up enough of the readahead window that we should start pulling in
 * more pages.
 */
void
page_cache_async_readahead(struct address_space *mapping,
                           struct file_ra_state *ra, struct file *filp,
                           struct page *page, pgoff_t offset,
                           unsigned long req_size)
{
        /* no read-ahead */
        if (!ra->ra_pages)
                return;

        /*
         * Same bit is used for PG_readahead and PG_reclaim.
         */
        if (PageWriteback(page))
                return;

        ClearPageReadahead(page);

        /*
         * Defer asynchronous read-ahead on IO congestion.
         */
        if (bdi_read_congested(mapping->backing_dev_info))
                return;

        /* do read-ahead */
        ondemand_readahead(mapping, ra, filp, true, offset, req_size);
}
```

当要读取的页在pagecache中,会执行页同步预读,page_cache_sync_readahead通用的文件同步预读实现,offset 表示要读取数据所在的起始页, req_size涉及的页数,其具体实现由ondemand_readahead完成.

## PageCache ReadAhead
