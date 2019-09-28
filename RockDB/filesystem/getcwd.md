# 系统调用getcwd

getcwd是获取进程工作空间的一个系统调用,该系统调用操作的主要数据结构有task_struct(内存中描述一个进程),fs_struct(描述进程的工作目录和根目录的结构),其中就维护这getcwd需要的信息,当前工作目录.

## fs_struct

```c
struct fs_struct {
        int users;
        rwlock_t lock;
        int umask;
        int in_exec;
        struct path root, pwd;
};
```

lock：用于表中字段的读/写自旋锁
umask：当打开文件设置文件权限时所使用的位掩码
root：根目录的目录项
pwd：当前工作目录的目录项

## getcwd实现

getcwd是获取进程当前工作目录,通过进程对象的fs->pwd dentry对象逐层向上查找路径,拼接出当前工作目录的绝对路径;
其函数首先申请一个page大小的内核空间,然后通过__d_path获取当前工作目录,再由copy_to_user拷贝到用户空间中,并是否内核申请的page空间;

```c
SYSCALL_DEFINE2(getcwd, char __user *, buf, unsigned long, size)
{
        int error;
        struct path pwd, root;
        char *page = (char *) __get_free_page(GFP_USER);

        if (!page)
                return -ENOMEM;

        read_lock(&current->fs->lock);
        pwd = current->fs->pwd;
        path_get(&pwd);
        root = current->fs->root;
        path_get(&root);
        read_unlock(&current->fs->lock);

        error = -ENOENT;
        spin_lock(&dcache_lock);
        if (!d_unlinked(pwd.dentry)) {
                unsigned long len;
                struct path tmp = root;
                char * cwd;

                cwd = __d_path(&pwd, &tmp, page, PAGE_SIZE);
                spin_unlock(&dcache_lock);

                error = PTR_ERR(cwd);
                if (IS_ERR(cwd))
                        goto out;

                error = -ERANGE;
                len = PAGE_SIZE + page - cwd;
                if (len <= size) {
                        error = len;
                        if (copy_to_user(buf, cwd, len))
                                error = -EFAULT;
                }
        } else
                spin_unlock(&dcache_lock);

out:
        path_put(&pwd);
        path_put(&root);
        free_page((unsigned long) page);
        return error;
}
```

## __d_path

__d_path根据提供的path和root,递归的找出并拼接出,path中的dentry对应的完整绝对路径;

```c
/**
 * __d_path - return the path of a dentry
 * @path: the dentry/vfsmount to report
 * @root: root vfsmnt/dentry (may be modified by this function)
 * @buffer: buffer to return value in
 * @buflen: buffer length
 *
 * Convert a dentry into an ASCII path name. If the entry has been deleted
 * the string " (deleted)" is appended. Note that this is ambiguous.
 *
 * Returns a pointer into the buffer or an error code if the
 * path was too long.
 *
 * "buflen" should be positive. Caller holds the dcache_lock.
 *
 * If path is not reachable from the supplied root, then the value of
 * root is changed (without modifying refcounts).
 */
char *__d_path(const struct path *path, struct path *root,
               char *buffer, int buflen)
{
        struct dentry *dentry = path->dentry;
        struct vfsmount *vfsmnt = path->mnt;
        char *end = buffer + buflen;
        char *retval;

        spin_lock(&vfsmount_lock);
        prepend(&end, &buflen, "\0", 1);
        if (d_unlinked(dentry) &&
                (prepend(&end, &buflen, " (deleted)", 10) != 0))
                        goto Elong;

        if (buflen < 1)
                goto Elong;
        /* Get '/' right */
        retval = end-1;
        *retval = '/';

        for (;;) {
                struct dentry * parent;
//递归的查找到"/"目录,终止查找
                if (dentry == root->dentry && vfsmnt == root->mnt)
                        break;
//如果当前dentry是root目录项或者是更目录的挂着点对应的目录项
                if (dentry == vfsmnt->mnt_root || IS_ROOT(dentry)) {
                        /* Global root? */ //如果是全局文件系统,则终止当前查找
                        if (vfsmnt->mnt_parent == vfsmnt) {
                                goto global_root;
                        }
//切换至上一层文件系统,继续查找,即当前文件系统的挂载点是别的文件系统的一个目录
                        dentry = vfsmnt->mnt_mountpoint;
                        vfsmnt = vfsmnt->mnt_parent;
                        continue;
                }
                parent = dentry->d_parent;
                prefetch(parent);
                if ((prepend_name(&end, &buflen, &dentry->d_name) != 0) ||
                    (prepend(&end, &buflen, "/", 1) != 0))
                        goto Elong;
                retval = end;
                dentry = parent;
        }

out:
        spin_unlock(&vfsmount_lock);
        return retval;

global_root:
        retval += 1;    /* hit the slash */
        if (prepend_name(&retval, &buflen, &dentry->d_name) != 0)
                goto Elong;
        root->mnt = vfsmnt;
        root->dentry = dentry;
        goto out;

Elong:
        retval = ERR_PTR(-ENAMETOOLONG);
        goto out;
}
```
