# fuse dev

## fuse 注册虚拟设备 fuse 设备类型 10

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
