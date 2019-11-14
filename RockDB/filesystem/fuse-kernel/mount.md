# 文件系统挂着

mount系统调用定义位于fs/namespace.c;

```c
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
                char __user *, type, unsigned long, flags, void __user *, data)
{
        int ret; 
        char *kernel_type;
        struct filename *kernel_dir;
        char *kernel_dev;
        unsigned long data_page;

        ret = copy_mount_string(type, &kernel_type);
        if (ret < 0) 
                goto out_type;

        kernel_dir = getname(dir_name);
        if (IS_ERR(kernel_dir)) {
                ret = PTR_ERR(kernel_dir);
                goto out_dir;
        }    

        ret = copy_mount_string(dev_name, &kernel_dev);
        if (ret < 0) 
                goto out_dev;

        ret = copy_mount_options(data, &data_page);
        if (ret < 0) 
                goto out_data;

        ret = do_mount(kernel_dev, kernel_dir->name, kernel_type, flags,
                (void *) data_page);

        free_page(data_page);
out_data:
        kfree(kernel_dev);
out_dev:
        putname(kernel_dir);
out_dir:
        kfree(kernel_type);
out_type:
        return ret; 
}

```

```c
/*
 * Flags is a 32-bit value that allows up to 31 non-fs dependent flags to
 * be given to the mount() call (ie: read-only, no-dev, no-suid etc).
 *
 * data is a (void *) that can point to any structure up to
 * PAGE_SIZE-1 bytes, which can contain arbitrary fs-dependent
 * information (or be NULL).
 *
 * Pre-0.97 versions of mount() didn't have a flags word.
 * When the flags word was introduced its top half was required
 * to have the magic value 0xC0ED, and this remained so until 2.4.0-test9.
 * Therefore, if this magic number is present, it carries no information
 * and must be discarded.
 */
long do_mount(const char *dev_name, const char *dir_name,
                const char *type_page, unsigned long flags, void *data_page)
{
        struct path path;
        int retval = 0;
        int mnt_flags = 0;

        /* Discard magic */
        if ((flags & MS_MGC_MSK) == MS_MGC_VAL)
                flags &= ~MS_MGC_MSK;

        /* Basic sanity checks */

        if (!dir_name || !*dir_name || !memchr(dir_name, 0, PAGE_SIZE))
                return -EINVAL;

        if (data_page)
                ((char *)data_page)[PAGE_SIZE - 1] = 0;

        /* ... and get the mountpoint */
        retval = kern_path(dir_name, LOOKUP_FOLLOW, &path);
        if (retval)
                return retval;

        retval = security_sb_mount(dev_name, &path,
                                   type_page, flags, data_page);
        if (!retval && !may_mount())
                retval = -EPERM;
        if (retval)
                goto dput_out;

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

        flags &= ~(MS_NOSUID | MS_NOEXEC | MS_NODEV | MS_ACTIVE | MS_BORN |
                   MS_NOATIME | MS_NODIRATIME | MS_RELATIME| MS_KERNMOUNT |
                   MS_STRICTATIME);

        if (flags & MS_REMOUNT)
                retval = do_remount(&path, flags & ~MS_REMOUNT, mnt_flags,
                                    data_page);
        else if (flags & MS_BIND)
                retval = do_loopback(&path, dev_name, flags & MS_REC);
        else if (flags & (MS_SHARED | MS_PRIVATE | MS_SLAVE | MS_UNBINDABLE))
                retval = do_change_type(&path, flags);
        else if (flags & MS_MOVE)
                retval = do_move_mount(&path, dev_name);
        else
                retval = do_new_mount(&path, type_page, flags, mnt_flags,
                                      dev_name, data_page);
dput_out:
        path_put(&path);
        return retval;
}

```

```c
/*
 * create a new mount for userspace and request it to be added into the
 * namespace's tree
 */
static int do_new_mount(struct path *path, const char *fstype, int flags,
                        int mnt_flags, const char *name, void *data)
{
        struct file_system_type *type;
        struct user_namespace *user_ns = current->nsproxy->mnt_ns->user_ns;
        struct vfsmount *mnt;
        int err;

        if (!fstype)
                return -EINVAL;

        type = get_fs_type(fstype);
        if (!type)
                return -ENODEV;

        if (user_ns != &init_user_ns) {
                if (!(type->fs_flags & FS_USERNS_MOUNT)) {
                        put_filesystem(type);
                        return -EPERM;
                }
                /* Only in special cases allow devices from mounts
                 * created outside the initial user namespace.
                 */
                if (!(type->fs_flags & FS_USERNS_DEV_MOUNT)) {
                        flags |= MS_NODEV;
                        mnt_flags |= MNT_NODEV;
                }
        }

        mnt = vfs_kern_mount(type, flags, name, data);
        if (!IS_ERR(mnt) && (type->fs_flags & FS_HAS_SUBTYPE) &&
            !mnt->mnt_sb->s_subtype)
                mnt = fs_set_subtype(mnt, fstype);

        put_filesystem(type);
        if (IS_ERR(mnt))
                return PTR_ERR(mnt);

        err = do_add_mount(real_mount(mnt), path, mnt_flags);
        if (err)
                mntput(mnt);
        return err;
}
```

```c
struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
        struct mount *mnt;
        struct dentry *root;

        if (!type)
                return ERR_PTR(-ENODEV);

        mnt = alloc_vfsmnt(name);
        if (!mnt)
                return ERR_PTR(-ENOMEM);

        if (flags & MS_KERNMOUNT)
                mnt->mnt.mnt_flags = MNT_INTERNAL;

        root = mount_fs(type, flags, name, data);
        if (IS_ERR(root)) {
                free_vfsmnt(mnt);
                return ERR_CAST(root);
        }

        mnt->mnt.mnt_root = root;
        mnt->mnt.mnt_sb = root->d_sb;
        mnt->mnt_mountpoint = mnt->mnt.mnt_root;
        mnt->mnt_parent = mnt;
        br_write_lock(&vfsmount_lock);
        list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
        br_write_unlock(&vfsmount_lock);
        return &mnt->mnt;
}
EXPORT_SYMBOL_GPL(vfs_kern_mount);

```

```c
struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
        struct dentry *root;
        struct super_block *sb; 
        char *secdata = NULL;
        int error = -ENOMEM;

        if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {
                secdata = alloc_secdata();
                if (!secdata)
                        goto out; 

                error = security_sb_copy_data(data, secdata);
                if (error)
                        goto out_free_secdata;
        }
        //根据挂载的文件系统类型,调用其自定义的mount方法,实现文件系统的挂着操作;
        root = type->mount(type, flags, name, data);
        if (IS_ERR(root)) {
                error = PTR_ERR(root);
                goto out_free_secdata;
        }   
        sb = root->d_sb;
        BUG_ON(!sb);
        WARN_ON(!sb->s_bdi);
        WARN_ON(sb->s_bdi == &default_backing_dev_info);
        sb->s_flags |= MS_BORN;

        error = security_sb_kern_mount(sb, flags, secdata);
        if (error)
                goto out_sb;

        /*   
         * filesystems should never set s_maxbytes larger than MAX_LFS_FILESIZE
         * but s_maxbytes was an unsigned long long for many releases. Throw
         * this warning for a little while to try and catch filesystems that
         * violate this rule.
         */
        WARN((sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
                "negative value (%lld)\n", type->name, sb->s_maxbytes);

        up_write(&sb->s_umount);
        free_secdata(secdata);
        return root;
out_sb:
        dput(root);
        deactivate_locked_super(sb);
out_free_secdata:
        free_secdata(secdata);
out:
        return ERR_PTR(error);
}
```