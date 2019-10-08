# mkdir

创建目录的系统调用陷入内核,会触发调用下面方法的调用:

SYSCALL_DEFINE2(mkdir, const char __user *, pathname, int, mode)
{
        return sys_mkdirat(AT_FDCWD, pathname, mode);
}
