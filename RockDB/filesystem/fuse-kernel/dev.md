# fuse dev

## 注册fuse模块

初始化fuse_conn_list链表;
初始化fuse文件系统,将fuse文件系统添加到file_systems链表中, 将fuseblk文件系统也添加到file_systems中;
初始化虚拟混杂设备,并注册到misc_list链表中;
创建fuse_kobj和connections_kobj两个内核对象;
初始化fusectl文件系统, 将fusectl文件系统添加到file_systems链表中;

```c
static int __init fuse_init(void)
{
        int res;

        printk(KERN_INFO "fuse init (API version %i.%i)\n",
               FUSE_KERNEL_VERSION, FUSE_KERNEL_MINOR_VERSION);

        INIT_LIST_HEAD(&fuse_conn_list);
        res = fuse_fs_init();
        if (res)
                goto err;

        res = fuse_dev_init();
        if (res)
                goto err_fs_cleanup;

        res = fuse_sysfs_init();
        if (res)
                goto err_dev_cleanup;

        res = fuse_ctl_init();
        if (res)
                goto err_sysfs_cleanup;

        sanitize_global_limit(&max_user_bgreq);
        sanitize_global_limit(&max_user_congthresh);

        return 0;

 err_sysfs_cleanup:
        fuse_sysfs_cleanup();
 err_dev_cleanup:
        fuse_dev_cleanup();
 err_fs_cleanup:
        fuse_fs_cleanup();
 err:
        return res;
}
```

创建内核对象fuse_kobj/connections_kobj;

```c
static struct kobject *fuse_kobj;
static struct kobject *connections_kobj;

static int fuse_sysfs_init(void)
{
        int err;

        fuse_kobj = kobject_create_and_add("fuse", fs_kobj);
        if (!fuse_kobj) {
                err = -ENOMEM;
                goto out_err;
        }

        connections_kobj = kobject_create_and_add("connections", fuse_kobj);
        if (!connections_kobj) {
                err = -ENOMEM;
                goto out_fuse_unregister;
        }

        return 0;

 out_fuse_unregister:
        kobject_put(fuse_kobj);
 out_err:
        return err;
}
```

卸载fuse模块

```c
static void __exit fuse_exit(void)
{
        printk(KERN_DEBUG "fuse exit\n");
  
        fuse_ctl_cleanup();
        fuse_sysfs_cleanup();
        fuse_fs_cleanup();
        fuse_dev_cleanup();
}
```

初始化fuse文件系统

```c
static struct file_system_type fuse_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "fuse",
        .fs_flags       = FS_HAS_SUBTYPE,
        .mount          = fuse_mount,
        .kill_sb        = fuse_kill_sb_anon,
};
MODULE_ALIAS_FS("fuse");

static int __init fuse_fs_init(void)
{
        int err;

        fuse_inode_cachep = kmem_cache_create("fuse_inode",
                                              sizeof(struct fuse_inode),
                                              0, SLAB_HWCACHE_ALIGN,
                                              fuse_inode_init_once);
        err = -ENOMEM;
        if (!fuse_inode_cachep)
                goto out;

        err = register_fuseblk();
        if (err)
                goto out2;

        err = register_filesystem(&fuse_fs_type);
        if (err)
                goto out3;

        return 0;

 out3:
        unregister_fuseblk();
 out2:
        kmem_cache_destroy(fuse_inode_cachep);
 out:
        return err;
}
```

卸载fuse文件系统

```c
static void fuse_fs_cleanup(void)
{
        unregister_filesystem(&fuse_fs_type);
        unregister_fuseblk();

        /*
         * Make sure all delayed rcu free inodes are flushed before we
         * destroy cache.
         */
        rcu_barrier();
        kmem_cache_destroy(fuse_inode_cachep);
}
```

注册和注销fuseblk文件系统

```c
static struct file_system_type fuseblk_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "fuseblk",
        .mount          = fuse_mount_blk,
        .kill_sb        = fuse_kill_sb_blk,
        .fs_flags       = FS_REQUIRES_DEV | FS_HAS_SUBTYPE,
};
MODULE_ALIAS_FS("fuseblk");

static inline int register_fuseblk(void)
{
        return register_filesystem(&fuseblk_fs_type);
}

static inline void unregister_fuseblk(void)
{
        unregister_filesystem(&fuseblk_fs_type);
}

```

## fuse 虚拟设备初始化

创建struct fuse_req 缓存对象, 注册fuse设备类型 10,misc_register创建虚拟混杂设备,并在内核中注册在misc_list中.

```c
const struct file_operations fuse_dev_operations = {
        .owner          = THIS_MODULE,
        .llseek         = no_llseek,
        .read           = do_sync_read,
        .aio_read       = fuse_dev_read,
        .splice_read    = fuse_dev_splice_read,
        .write          = do_sync_write,
        .aio_write      = fuse_dev_write,
        .splice_write   = fuse_dev_splice_write,
        .poll           = fuse_dev_poll,
        .release        = fuse_dev_release,
        .fasync         = fuse_dev_fasync,
};
EXPORT_SYMBOL_GPL(fuse_dev_operations);

static struct miscdevice fuse_miscdevice = {
        .minor = FUSE_MINOR,
        .name  = "fuse",
        .fops = &fuse_dev_operations,
};
```

```c
int __init fuse_dev_init(void)
{
        int err = -ENOMEM;
        fuse_req_cachep = kmem_cache_create("fuse_request",
                                            sizeof(struct fuse_req),
                                            0, 0, NULL);
        if (!fuse_req_cachep)
                goto out;

        err = misc_register(&fuse_miscdevice);
        if (err)
                goto out_cache_clean;

        return 0;
  
 out_cache_clean:
        kmem_cache_destroy(fuse_req_cachep);
 out:
        return err;
}
```

注销fuse虚拟设备,并释放fuse_req_cachep指向的缓存对象

```c
void fuse_dev_cleanup(void)
{
        misc_deregister(&fuse_miscdevice);
        kmem_cache_destroy(fuse_req_cachep); 
}
```

注册fusectl文件系统类型到

```c
static struct file_system_type fuse_ctl_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "fusectl",
        .mount          = fuse_ctl_mount,
        .kill_sb        = fuse_ctl_kill_sb,
};
MODULE_ALIAS_FS("fusectl");
```

```c
int __init fuse_ctl_init(void)
{
        return register_filesystem(&fuse_ctl_fs_type);
}
```

注销fusectl文件系统类型

```c
void fuse_ctl_cleanup(void)
{
        unregister_filesystem(&fuse_ctl_fs_type);
}
```

