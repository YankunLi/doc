# Fuse Open

fuse open可分成两部分，一部分是fuse文件系统的内核实现部分, 主要是将文件打开请求封装成fuse_req，提交给用户态文件系统处理，另一部分是用户态文件系统的实现部分，是文件系统相关操作的实现。本文主要分析第一部分。
fuse文件系统中实现open的函数是：fuse_open
```
static int fuse_open(struct inode *inode, struct file *file)
{
        return fuse_open_common(inode, file, false);
}
```

fuse_open直接调用了fuse_open_common,将参数inode，file传入，并指明文件类型：false，不是目录。
```
int fuse_open_common(struct inode *inode, struct file *file, bool isdir)
{
        struct fuse_mount *fm = get_fuse_mount(inode);
        struct fuse_conn *fc = fm->fc;
        int err;
        bool is_wb_truncate = (file->f_flags & O_TRUNC) &&
                          fc->atomic_o_trunc &&
                          fc->writeback_cache;
        bool dax_truncate = (file->f_flags & O_TRUNC) &&
                          fc->atomic_o_trunc && FUSE_IS_DAX(inode);

        if (fuse_is_bad(inode))
                return -EIO;

        err = generic_file_open(inode, file);
        if (err)
                return err;

        if (is_wb_truncate || dax_truncate) {
                inode_lock(inode);
                fuse_set_nowrite(inode);
        }

        if (dax_truncate) {
                down_write(&get_fuse_inode(inode)->i_mmap_sem);
                err = fuse_dax_break_layouts(inode, 0, 0);
                if (err)
                        goto out;
        }

        err = fuse_do_open(fm, get_node_id(inode), file, isdir);
        if (!err)
                fuse_finish_open(inode, file);

out:
        if (dax_truncate)
                up_write(&get_fuse_inode(inode)->i_mmap_sem);

        if (is_wb_truncate | dax_truncate) {
                fuse_release_nowrite(inode);
                inode_unlock(inode);
        }

        return err;
}
```
- 该函数通过inode获取fuse_mount,该结构包含了fuse的挂载信息，其中就包括fuse_conn，代表内核态和用户态进程的连接。
- fc->writeback_cache标识fuse文件系统的IO模式，有writeback和writethrough（默认模式)。
- 该函数先检查了请求参数，然后调用核心的函数fuse_do_open：执行fuse文件打开操作；fuse_finish_open: 执行fuse打开完成后的处理工作。

## fuse_do_open:
```
int fuse_do_open(struct fuse_mount *fm, u64 nodeid, struct file *file,
                 bool isdir)
{
        struct fuse_file *ff = fuse_file_open(fm, nodeid, file->f_flags, isdir);

        if (!IS_ERR(ff))
                file->private_data = ff;

        return PTR_ERR_OR_ZERO(ff);
}
EXPORT_SYMBOL_GPL(fuse_do_open);
```
fuse_do_open调用fuse_file_open,返回的 fuse_file实例，作为file->private_data的值。

```
struct fuse_file *fuse_file_open(struct fuse_mount *fm, u64 nodeid,
                                 unsigned int open_flags, bool isdir)
{
        struct fuse_conn *fc = fm->fc;
        struct fuse_file *ff;
        int opcode = isdir ? FUSE_OPENDIR : FUSE_OPEN; //初始化操作类型: 打开文件和打开目录是两种不同的操作。

        ff = fuse_file_alloc(fm); //分配fuse_file实例。
        if (!ff)
                return ERR_PTR(-ENOMEM);

        ff->fh = 0;
        /* Default for no-open */
        ff->open_flags = FOPEN_KEEP_CACHE | (isdir ? FOPEN_CACHE_DIR : 0);
        if (isdir ? !fc->no_opendir : !fc->no_open) { //判断是否实现了open函数。
                struct fuse_open_out outarg; //用于保存发送open请求的结果。
                int err;

                err = fuse_send_open(fm, nodeid, open_flags, opcode, &outarg);
                if (!err) {
                        ff->fh = outarg.fh;
                        ff->open_flags = outarg.open_flags;

                } else if (err != -ENOSYS) {
                        fuse_file_free(ff);
                        return ERR_PTR(err);
                } else {
                        if (isdir)
                                fc->no_opendir = 1;
                        else
                                fc->no_open = 1;
                }
        }

        if (isdir)
                ff->open_flags &= ~FOPEN_DIRECT_IO;

        ff->nodeid = nodeid;

        return ff;
}
```
- 该函数主要是确定操作类型，分配fuse_file实例，进一步调用fuse_send_open，执行fuse打开操作。

```
static int fuse_send_open(struct fuse_mount *fm, u64 nodeid,
                          unsigned int open_flags, int opcode,
                          struct fuse_open_out *outargp)
{
        struct fuse_open_in inarg;
        FUSE_ARGS(args);//定义类型fuse_args 的请求参数。

        memset(&inarg, 0, sizeof(inarg));
        inarg.flags = open_flags & ~(O_CREAT | O_EXCL | O_NOCTTY);
        if (!fm->fc->atomic_o_trunc)
                inarg.flags &= ~O_TRUNC;

        if (fm->fc->handle_killpriv_v2 &&
            (inarg.flags & O_TRUNC) && !capable(CAP_FSETID)) {
                inarg.open_flags |= FUSE_OPEN_KILL_SUIDGID;
        }

        // 初始化请求参数。
        args.opcode = opcode;
        args.nodeid = nodeid;
        args.in_numargs = 1;
        args.in_args[0].size = sizeof(inarg);
        args.in_args[0].value = &inarg;
        args.out_numargs = 1;
        args.out_args[0].size = sizeof(*outargp);
        args.out_args[0].value = outargp;

        return fuse_simple_request(fm, &args);
}

该函数通过 fuse_simple_request函数向用户态发送请求，请求参数在fuse_args 参数实例中。

### fuse_simple_request
```
```
ssize_t fuse_simple_request(struct fuse_mount *fm, struct fuse_args *args)
{
        struct fuse_conn *fc = fm->fc;
        struct fuse_req *req;
        ssize_t ret;

        if (args->force) {
                atomic_inc(&fc->num_waiting);
                req = fuse_request_alloc(fm, GFP_KERNEL | __GFP_NOFAIL); //分配fuse_req实例

                if (!args->nocreds)
                        fuse_force_creds(req);

                __set_bit(FR_WAITING, &req->flags); //设置fuse_req状态。
                __set_bit(FR_FORCE, &req->flags);
        } else {
                WARN_ON(args->nocreds);
                req = fuse_get_req(fm, false);
                if (IS_ERR(req))
                        return PTR_ERR(req);
        }

        /* Needs to be done after fuse_get_req() so that fc->minor is valid */
        fuse_adjust_compat(fc, args);
        fuse_args_to_req(req, args);

        if (!args->noreply)
                __set_bit(FR_ISREPLY, &req->flags);
        __fuse_request_send(req); //发送请求。
        ret = req->out.h.error;
        if (!ret && args->out_argvar) {
                BUG_ON(args->out_numargs == 0);
                ret = args->out_args[args->out_numargs - 1].size;
        }
        fuse_put_request(req); //释放fuse_req

        return ret;
}
```
- 该函数主是根据请求参数，构造fuse_req实例。然后调用__fuse_request_send，发送该请求。此时fuse_req的状态：FR_PENDDING, FR_WAITING, FR_FORCE。

```
static void __fuse_request_send(struct fuse_req *req)
{
        struct fuse_iqueue *fiq = &req->fm->fc->iq; //获取与用户态进程连接的请求队列。

        BUG_ON(test_bit(FR_BACKGROUND, &req->flags));
        spin_lock(&fiq->lock);
        if (!fiq->connected) {
                spin_unlock(&fiq->lock);
                req->out.h.error = -ENOTCONN;
        } else {
                req->in.h.unique = fuse_get_unique(fiq); //为请求分配唯一的请求id。
                /* acquire extra reference, since request is still needed
                   after fuse_request_end() */
                __fuse_get_request(req);
                queue_request_and_unlock(fiq, req); //将请求放入到连接的请求队列中。

                request_wait_answer(req); //等待请求的响应。
                /* Pairs with smp_wmb() in fuse_request_end() */
                smp_rmb();
        }
}
```
- 该函数首先获取与用户态进程连接的请求队列，然后将请求放入请求队列中(queue_request_and_unlock)，最后等待该请求的响应结果(request_wait_answer)。

```
static void queue_request_and_unlock(struct fuse_iqueue *fiq,
                                     struct fuse_req *req)
__releases(fiq->lock)
{
        req->in.h.len = sizeof(struct fuse_in_header) +
                fuse_len_args(req->args->in_numargs,
                              (struct fuse_arg *) req->args->in_args);
        list_add_tail(&req->list, &fiq->pending); // 将请求放入到请求队列中
        fiq->ops->wake_pending_and_unlock(fiq); //唤醒等着在该队列上的处理单元可以处理任务了。
}
```
- 该函数将请求放入请求队列的pending队列中，然后唤醒阻塞在该队列上的任务，通知其有任务来了，可以处理了。

```
static void request_wait_answer(struct fuse_req *req)
{
        struct fuse_conn *fc = req->fm->fc;
        struct fuse_iqueue *fiq = &fc->iq;
        int err;

        if (!fc->no_interrupt) {
                /* Any signal may interrupt this */
                err = wait_event_interruptible(req->waitq,
                                        test_bit(FR_FINISHED, &req->flags));
                if (!err)
                        return;

                set_bit(FR_INTERRUPTED, &req->flags);
                /* matches barrier in fuse_dev_do_read() */
                smp_mb__after_atomic();
                if (test_bit(FR_SENT, &req->flags))
                        queue_interrupt(req);
        }

        if (!test_bit(FR_FORCE, &req->flags)) {
                /* Only fatal signals may interrupt this */
                err = wait_event_killable(req->waitq,
                                        test_bit(FR_FINISHED, &req->flags));
                if (!err)
                        return;

                spin_lock(&fiq->lock);
                /* Request is not yet in userspace, bail out */
                if (test_bit(FR_PENDING, &req->flags)) {
                        list_del(&req->list);
                        spin_unlock(&fiq->lock);
                        __fuse_put_request(req);
                        req->out.h.error = -EINTR;
                        return;
                }
                spin_unlock(&fiq->lock);
        }

        /*
         * Either request is already in userspace, or it was forced.
         * Wait it out.
         */
        wait_event(req->waitq, test_bit(FR_FINISHED, &req->flags));
}
```
- 该函数先判断fuse是否实现了中断函数，如果实现了，就使用wait_event_interruptib可中断的阻塞在fuse_req->waitq上。
- 如果req-flags 设置了FR_FORCE，则使用wait_event_killable，可变kill可被faltal signal信号中断的阻塞在fuse_req->waitq上。
- wait_event 监听在fuse_req->waitq上等待请求返回结果, 切不可中断。 如果请求返回会回到fuse_do_open处。

## fuse_finish_open:
```
void fuse_finish_open(struct inode *inode, struct file *file)
{
        struct fuse_file *ff = file->private_data;
        struct fuse_conn *fc = get_fuse_conn(inode);

        if (!(ff->open_flags & FOPEN_KEEP_CACHE))
                invalidate_inode_pages2(inode->i_mapping);
        if (ff->open_flags & FOPEN_STREAM)
                stream_open(inode, file);
        else if (ff->open_flags & FOPEN_NONSEEKABLE)
                nonseekable_open(inode, file);
        if (fc->atomic_o_trunc && (file->f_flags & O_TRUNC)) {
                struct fuse_inode *fi = get_fuse_inode(inode);

                spin_lock(&fi->lock);
                fi->attr_version = atomic64_inc_return(&fc->attr_version);
                i_size_write(inode, 0);
                spin_unlock(&fi->lock);
                fuse_invalidate_attr(inode);
                if (fc->writeback_cache)
                        file_update_time(file);
        }
        if ((file->f_mode & FMODE_WRITE) && fc->writeback_cache)
                fuse_link_write_file(file);
}
```

-  该函数是对fuse open结果的相关处理。

# 总结：
用户态进程，发起open请求，fuse接收到该请求后，将open请求信息封装到fuse_req中，然后把该fuse_req放入连接的pending队列中，该连接是fuse内核部分与用户态文件系统互联的通道。
然后等待该请求被另外一个处理单元处理并返回处理结果。另外一个处理单元实际只是把请求转交给用户天文件系统，然后把用户态文件系统处理的结果再转给请求fuse_req。
