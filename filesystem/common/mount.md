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

        ret = user_path_at(AT_FDCWD, dir_name, LOOKUP_FOLLOW, &path); //检查是否存在，以及相关权限是否允许, 并创建path。
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
                //该函数会调用vfs_get_super，分配super_block, 并调用fuse_fill_super，来设置super_block变量。同时创建根目录的dentry，inode。
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
