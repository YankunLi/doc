# filesystem mount 函数解析 以fuse挂载为例：

mount系统调用定义在fs/namespace.c文件中。

dev_name: 文件系统挂载的设备，有些文件系统不需要，该值可以为空。
dir_name: 文件系统被挂载上的目录，即挂载点。
type： 文件系统类型。
flags：挂载文件系统的一些特性设置。相关取值定义在include/linux/fs.h。
data：用户自定义的数。
```
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
                char __user *, type, unsigned long, flags, void __user *, data)
{
        int ret;
        char *kernel_type;
        char *kernel_dev;
        void *options;

        kernel_type = copy_mount_string(type);// 将用户态的变量
        ret = PTR_ERR(kernel_type);
        if (IS_ERR(kernel_type))
                goto out_type;

        kernel_dev = copy_mount_string(dev_name);
        ret = PTR_ERR(kernel_dev);
        if (IS_ERR(kernel_dev))
                goto out_dev;

        options = copy_mount_options(data);
        ret = PTR_ERR(options);
        if (IS_ERR(options))
                goto out_data;

        ret = do_mount(kernel_dev, dir_name, kernel_type, flags, options);

        kfree(options);
out_data:
        kfree(kernel_dev);
out_dev:
        kfree(kernel_type);
out_type:
        return ret;
}
```
该函数主要是将用户态传递的变量，拷贝到内核态来，然后调用 do_mount 使用，结束后释放对应的内核空间.

```
long do_mount(const char *dev_name, const char __user *dir_name,
                const char *type_page, unsigned long flags, void *data_page)
{
        struct path path;
        int ret;

        ret = user_path_at(AT_FDCWD, dir_name, LOOKUP_FOLLOW, &path); //基于挂载目录dir_name 创建struct path。
        // 检查是否存在，以及相关权限是否允许, 并创建path。
        if (ret)
                return ret;
        ret = path_mount(dev_name, &path, type_page, flags, data_page);
        path_put(&path);
        return ret;
}
```
该函数检查挂载点目录是否存在，并为其创建struct path，然后继续调用path_mount.

```
int path_mount(const char *dev_name, struct path *path,
                const char *type_page, unsigned long flags, void *data_page)
{
        unsigned int mnt_flags = 0, sb_flags;
        int ret;

        /* Discard magic */
        if ((flags & MS_MGC_MSK) == MS_MGC_VAL)
                flags &= ~MS_MGC_MSK;

        /* Basic sanity checks */
        if (data_page)
                ((char *)data_page)[PAGE_SIZE - 1] = 0;

        if (flags & MS_NOUSER)
                return -EINVAL;

        ret = security_sb_mount(dev_name, path, type_page, flags, data_page); // 对挂载前做安全检查。
        if (ret)
                return ret;
        if (!may_mount())
                return -EPERM;
        if ((flags & SB_MANDLOCK) && !may_mandlock())
                return -EPERM;

        /* Default to relatime unless overriden */
        if (!(flags & MS_NOATIME))
                mnt_flags |= MNT_RELATIME;

        /* Separate the per-mountpoint flags */
        if (flags & MS_NOSUID)
                mnt_flags |= MNT_NOSUID;
        if (flags & MS_NODEV)
                mnt_flags |= MNT_NODEV;
        if (flags & MS_NOEXEC)
                mnt_flags |= MNT_NOEXEC;
        if (flags & MS_NOATIME)
                mnt_flags |= MNT_NOATIME;
        if (flags & MS_NODIRATIME)
                mnt_flags |= MNT_NODIRATIME;
        if (flags & MS_STRICTATIME)
                mnt_flags &= ~(MNT_RELATIME | MNT_NOATIME);
        if (flags & MS_RDONLY)
                mnt_flags |= MNT_READONLY;
        if (flags & MS_NOSYMFOLLOW)
                mnt_flags |= MNT_NOSYMFOLLOW;

        /* The default atime for remount is preservation */
        if ((flags & MS_REMOUNT) &&
            ((flags & (MS_NOATIME | MS_NODIRATIME | MS_RELATIME |
                       MS_STRICTATIME)) == 0)) {
                mnt_flags &= ~MNT_ATIME_MASK;
                mnt_flags |= path->mnt->mnt_flags & MNT_ATIME_MASK;
        }

        sb_flags = flags & (SB_RDONLY |
                            SB_SYNCHRONOUS |
                            SB_MANDLOCK |
                            SB_DIRSYNC |
                            SB_SILENT |
                            SB_POSIXACL |
                            SB_LAZYTIME |
                            SB_I_VERSION);

        if ((flags & (MS_REMOUNT | MS_BIND)) == (MS_REMOUNT | MS_BIND))
                return do_reconfigure_mnt(path, mnt_flags);
        if (flags & MS_REMOUNT)
                return do_remount(path, flags, sb_flags, mnt_flags, data_page);
        if (flags & MS_BIND)
                return do_loopback(path, dev_name, flags & MS_REC);
        if (flags & (MS_SHARED | MS_PRIVATE | MS_SLAVE | MS_UNBINDABLE))
                return do_change_type(path, flags);
        if (flags & MS_MOVE)
                return do_move_mount_old(path, dev_name);

        return do_new_mount(path, type_page, sb_flags, mnt_flags, dev_name,
                            data_page);
}
```
该函数首先参数及操作的检查，包括参数检查和mount操作的安全性，然后从flags中分离出mnt_flags和sb_flags，再根据flags参数选择不同的执行函数：do_reconfigure_mnt，do_remount，do_loopback，do_change_type, do_move_mount_old, do_new_mount。这里我们只分析do_new_mount，创建一个新的挂载。

```
static int do_new_mount(struct path *path, const char *fstype, int sb_flags,
                        int mnt_flags, const char *name, void *data)
{
        struct file_system_type *type;
        struct fs_context *fc;
        const char *subtype = NULL;
        int err = 0;

        if (!fstype)
                return -EINVAL;

        type = get_fs_type(fstype); //根据fstype获取文件系统类型实例变量。
        if (!type)
                return -ENODEV;

        if (type->fs_flags & FS_HAS_SUBTYPE) {
                subtype = strchr(fstype, '.');
                if (subtype) {
                        subtype++;
                        if (!*subtype) {
                                put_filesystem(type);
                                return -EINVAL;
                        }
                }
        }

        fc = fs_context_for_mount(type, sb_flags); // 分配并设置本次挂载操作需要的vfs上下文(struct fs_context)实例和fuse上下文(struct fuse_fs_context)实例。
        //由文件系统type获取fuse文件系统实例，调用其init_fs_context函数钩子(对应fuse_init_fs_context函数)，分配并初始化fuse文件系统的上下文。
        //struct fs_context 属性fs_private 指向fuse的文件系统上下文变量, 属性ops(struct fs_context_operations) 指向fuse文件系统提供的操作函数变量。
        put_filesystem(type);
        if (IS_ERR(fc))
                return PTR_ERR(fc);

        if (subtype)
                err = vfs_parse_fs_string(fc, "subtype",
                                          subtype, strlen(subtype)); //解析参数，赋值fc中相关变量。
                //fs_context通过操作函数（ops）的 .parse_param,该函数指向fuse文件系统 fuse_parse_param, 解析参数，设置fuse上下文属性。
        if (!err && name)
                err = vfs_parse_fs_string(fc, "source", name, strlen(name));
        if (!err)
                err = parse_monolithic_mount_data(fc, data); //解析data中有格式的参数，然后再调用vfs_parse_fs_string,设置fc中相关变量。
        if (!err && !mount_capable(fc))
                err = -EPERM;
        if (!err)
                err = vfs_get_tree(fc); // 该函数会调用fs_context 操作对象ops的get_tree函数, 实际调用fuse_get_tree。
                //该函数会调用vfs_get_super，分配super_block, 并调用fuse_fill_super，来设置super_block变量。同时创建挂载目录的dentry，inode。
                //dentry的d_inode指向对应的inode。super_block的s_root指向dentry。
                //该函数中也初始化并实例化了fuse_conn fuse_mount,super_block的s_fs_info指向fuse_mount, fuse_mount可以索引到fuse_conn。
                //super_block的s_d_op被指向fuse_dentry_operations变量。fuse_conn会交给fusectl文件系统管理。
        if (!err)
                err = do_new_mount_fc(fc, path, mnt_flags);

        put_fs_context(fc); //释放掉文件系统上下文。
        return err;
}
```
fuse 文件系统类型常量实例
```
static struct file_system_type fuse_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "fuse",
        .fs_flags       = FS_HAS_SUBTYPE | FS_USERNS_MOUNT,
        .init_fs_context = fuse_init_fs_context,// 用于分配并设置fuse 文件系统上下文(struct fuse_fs_context), 同时设置fs_context变量的ops( struct fs_context_operations)和fs_private(void *)该变量指向fuse文件系统上下文（struct fuse_fs_context);
        .parameters     = fuse_fs_parameters,
        .kill_sb        = fuse_kill_sb_anon,
};
```
fuse 文件系统上下文变量操作函数变量。
```
static const struct fs_context_operations fuse_context_ops = {
        .free           = fuse_free_fc,
        .parse_param    = fuse_parse_param,
        .reconfigure    = fuse_reconfigure,
        .get_tree       = fuse_get_tree, //设置super_block, 创建更目录的dentry，inode，以及fuse中的相关变量fuse_conn, fuse_mount。
};
```

struct fuse_conn可供多个文件系统使用,其属性struct list_head mounts 维护一组使用该连接的文件系统。
struct super_block->s_fs_info(struct fuse_mount *) struct fuse_mount->fc( struct fuse_conn *)
                                                                    ->sb(struct super_block *)

```
/*
 * Create a new mount using a superblock configuration and request it
 * be added to the namespace tree.
 */
static int do_new_mount_fc(struct fs_context *fc, struct path *mountpoint,
                           unsigned int mnt_flags)
{
        struct vfsmount *mnt;
        struct mountpoint *mp;
        struct super_block *sb = fc->root->d_sb;
        int error;

        error = security_sb_kern_mount(sb);
        if (!error && mount_too_revealing(sb, &mnt_flags))
                error = -EPERM;

        if (unlikely(error)) {
                fc_drop_locked(fc);
                return error;
        }

        up_write(&sb->s_umount);

        mnt = vfs_create_mount(fc); //创建文件系统的挂载对象struct mount。
        if (IS_ERR(mnt))
                return PTR_ERR(mnt);

        mnt_warn_timestamp_expiry(mountpoint, mnt);

        mp = lock_mount(mountpoint);
        if (IS_ERR(mp)) {
                mntput(mnt);
                return PTR_ERR(mp);
        }
        error = do_add_mount(real_mount(mnt), mp, mountpoint, mnt_flags);
        unlock_mount(mp);
        if (error < 0)
                mntput(mnt);
        return error;
}
```
该函数主要调用了vfs_create_mount创建struct mount(vfsmount 是其属性)，lock_mount创建mountpoint对象。
最后使用vfsmount和lock_mount作为参数调用do_add_mount。


```
/**
 * vfs_create_mount - Create a mount for a configured superblock
 * @fc: The configuration context with the superblock attached
 *
 * Create a mount to an already configured superblock.  If necessary, the
 * caller should invoke vfs_get_tree() before calling this.
 *
 * Note that this does not attach the mount to anything.
 */
struct vfsmount *vfs_create_mount(struct fs_context *fc)
{
        struct mount *mnt;

        if (!fc->root)
                return ERR_PTR(-EINVAL);

        mnt = alloc_vfsmnt(fc->source ?: "none");
        if (!mnt)
                return ERR_PTR(-ENOMEM);

        if (fc->sb_flags & SB_KERNMOUNT)
                mnt->mnt.mnt_flags = MNT_INTERNAL;

        atomic_inc(&fc->root->d_sb->s_active);
        mnt->mnt.mnt_sb         = fc->root->d_sb;
        mnt->mnt.mnt_root       = dget(fc->root);
        mnt->mnt_mountpoint     = mnt->mnt.mnt_root;
        mnt->mnt_parent         = mnt;

        lock_mount_hash();
        list_add_tail(&mnt->mnt_instance, &mnt->mnt.mnt_sb->s_mounts);
        unlock_mount_hash();
        return &mnt->mnt;
}
EXPORT_SYMBOL(vfs_create_mount);
```
为新建的super_block创建struct vfsmount:描述的是一个独立文件系统的挂载信息，每个不同挂载点对应一个独立的vfsmount结构，
属于同一文件系统的所有目录和文件隶属于同一个vfsmount，该vfsmount结构对应于该文件系统顶层目录，即挂载目录。ref[https://www.cnblogs.com/Wandererzj/archive/2012/04/12/2444888.html]

```
static struct mountpoint *lock_mount(struct path *path)
{
        struct vfsmount *mnt;
        struct dentry *dentry = path->dentry;
retry:
        inode_lock(dentry->d_inode);
        if (unlikely(cant_mount(dentry))) {
                inode_unlock(dentry->d_inode);
                return ERR_PTR(-ENOENT);
        }
        namespace_lock();
        mnt = lookup_mnt(path);
        if (likely(!mnt)) {
                struct mountpoint *mp = get_mountpoint(dentry); //为挂载目录创建mountpoint。
                if (IS_ERR(mp)) {
                        namespace_unlock();
                        inode_unlock(dentry->d_inode);
                        return mp;
                }
                return mp;
        }
        namespace_unlock();
        inode_unlock(path->dentry->d_inode);
        path_put(path);
        path->mnt = mnt;
        dentry = path->dentry = dget(mnt->mnt_root);
        goto retry;
}
```
该函数获取/或创建path所对应的mountpoint对象。

```
/*
 * add a mount into a namespace's mount tree
 */
static int do_add_mount(struct mount *newmnt, struct mountpoint *mp,
                        struct path *path, int mnt_flags)
{
        struct mount *parent = real_mount(path->mnt); //获取挂载目录所在文件系统的struct mount。

        mnt_flags &= ~MNT_INTERNAL_FLAGS;

        if (unlikely(!check_mnt(parent))) {
                /* that's acceptable only for automounts done in private ns */
                if (!(mnt_flags & MNT_SHRINKABLE))
                        return -EINVAL;
                /* ... and for those we'd better have mountpoint still alive */
                if (!parent->mnt_ns)
                        return -EINVAL;
        }

        /* Refuse the same filesystem on the same mount point */
        if (path->mnt->mnt_sb == newmnt->mnt.mnt_sb &&
            path->mnt->mnt_root == path->dentry)
                return -EBUSY;

        if (d_is_symlink(newmnt->mnt.mnt_root))
                return -EINVAL;

        newmnt->mnt.mnt_flags = mnt_flags;
        return graft_tree(newmnt, parent, mp);
}
```
该函数意图是将心的mount添加到挂载树的命名空间中，核心操作在graft_tree中实现，其他只是检查新的mount 对象和老的mount相关信息。
```
static int graft_tree(struct mount *mnt, struct mount *p, struct mountpoint *mp)
{
        if (mnt->mnt.mnt_sb->s_flags & SB_NOUSER)
                return -EINVAL;

        if (d_is_dir(mp->m_dentry) !=
              d_is_dir(mnt->mnt.mnt_root))
                return -ENOTDIR;

        return attach_recursive_mnt(mnt, p, mp, false);
}
```
graft_tree主要是调用attach_recursive_mnt 函数。


```
static int attach_recursive_mnt(struct mount *source_mnt,
                        struct mount *dest_mnt,
                        struct mountpoint *dest_mp,
                        bool moving)
{
        struct user_namespace *user_ns = current->nsproxy->mnt_ns->user_ns;
        HLIST_HEAD(tree_list);
        struct mnt_namespace *ns = dest_mnt->mnt_ns;
        struct mountpoint *smp;
        struct mount *child, *p;
        struct hlist_node *n;
        int err;

        /* Preallocate a mountpoint in case the new mounts need
         * to be tucked under other mounts.
         */
        smp = get_mountpoint(source_mnt->mnt.mnt_root); //获取员mount的mountpoint。
        if (IS_ERR(smp))
                return PTR_ERR(smp);

        /* Is there space to add these mounts to the mount namespace? */
        if (!moving) {
                err = count_mounts(ns, source_mnt);
                if (err)
                        goto out;
        }

        if (IS_MNT_SHARED(dest_mnt)) {
                err = invent_group_ids(source_mnt, true);
                if (err)
                        goto out;
                err = propagate_mnt(dest_mnt, dest_mp, source_mnt, &tree_list);
                lock_mount_hash();
                if (err)
                        goto out_cleanup_ids;
                for (p = source_mnt; p; p = next_mnt(p, source_mnt))
                        set_mnt_shared(p);
        } else {
                lock_mount_hash();
        }
        if (moving) {
                unhash_mnt(source_mnt);
                attach_mnt(source_mnt, dest_mnt, dest_mp);
                touch_mnt_namespace(source_mnt->mnt_ns);
        } else {
                if (source_mnt->mnt_ns) {
                        /* move from anon - the caller will destroy */
                        list_del_init(&source_mnt->mnt_ns->list);
                }
                mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt); // 设置源mount。
                commit_tree(source_mnt); //挂载源mount。
        }

        hlist_for_each_entry_safe(child, n, &tree_list, mnt_hash) {
                struct mount *q;
                hlist_del_init(&child->mnt_hash);
                q = __lookup_mnt(&child->mnt_parent->mnt,
                                 child->mnt_mountpoint);
                if (q)
                        mnt_change_mountpoint(child, smp, q);
                /* Notice when we are propagating across user namespaces */
                if (child->mnt_parent->mnt_ns->user_ns != user_ns)
                        lock_mnt_tree(child);
                child->mnt.mnt_flags &= ~MNT_LOCKED;
                commit_tree(child);
        }
        put_mountpoint(smp);
        unlock_mount_hash();

        return 0;

 out_cleanup_ids:
        while (!hlist_empty(&tree_list)) {
                child = hlist_entry(tree_list.first, struct mount, mnt_hash);
                child->mnt_parent->mnt_ns->pending_mounts = 0;
                umount_tree(child, UMOUNT_SYNC);
        }
        unlock_mount_hash();
        cleanup_group_ids(source_mnt, NULL);
 out:
        ns->pending_mounts = 0;

        read_seqlock_excl(&mount_lock);
        put_mountpoint(smp);
        read_sequnlock_excl(&mount_lock);

        return err;
}
```
该函数主要通过 mnt_set_mountpoint和commit_tree, 将源mount挂载到mount_hashtable中。

```
/*
 * vfsmount lock must be held for write
 */
void mnt_set_mountpoint(struct mount *mnt,
                        struct mountpoint *mp,
                        struct mount *child_mnt)
{
        mp->m_count++;
        mnt_add_count(mnt, 1);  /* essentially, that's mntget */
        child_mnt->mnt_mountpoint = mp->m_dentry; // 源mount的mnt_mountpoint指向挂载点的dentry。
        child_mnt->mnt_parent = mnt; //源mount的mnt_parent指向目标mount。
        child_mnt->mnt_mp = mp; //源mount的mnt_mp指向挂载点。
        hlist_add_head(&child_mnt->mnt_mp_list, &mp->m_list); //把源mount连接到挂载点的m_list链表上。
}
```
// 设置源mount的挂载点的dentry mnt_mountpoint, 目标挂载点mnt_parent, 源挂载点的关注点mnt_mp。

```
/*
 * vfsmount lock must be held for write
 */
static void commit_tree(struct mount *mnt)
{
        struct mount *parent = mnt->mnt_parent;
        struct mount *m;
        LIST_HEAD(head);
        struct mnt_namespace *n = parent->mnt_ns;

        BUG_ON(parent == mnt);

        list_add_tail(&head, &mnt->mnt_list);
        list_for_each_entry(m, &head, mnt_list)
                m->mnt_ns = n;

        list_splice(&head, n->list.prev);

        n->mounts += n->pending_mounts;
        n->pending_mounts = 0;

        __attach_mnt(mnt, parent); //将源mount 挂载到mount_hashtable中<目标vfsmount, 挂载点dentry> -> 源mount(通过mnt_hash连接)
        touch_mnt_namespace(n);
}
```
该函数将源mount挂载到mount_hashtable中, key是<目标vfsmount，挂载点dentry>。
```
static void __attach_mnt(struct mount *mnt, struct mount *parent)
{
        hlist_add_head_rcu(&mnt->mnt_hash,
                           m_hash(&parent->mnt, mnt->mnt_mountpoint));
        list_add_tail(&mnt->mnt_child, &parent->mnt_mounts);
}
```
将源mount连接到mount_hashtable<目标vfsmount, 挂载点dentry>对应的链表中，并将源mount通过mnt_child，连接到目标mount的mnt_mounts链表中。
