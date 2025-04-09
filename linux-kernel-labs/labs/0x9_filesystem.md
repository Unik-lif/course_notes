## 实验：文件系统
Linux本身支持50多种文件系统，而系统所支持的文件系统将会以模块的方式来进行移植，会利用一个`struct file_system_type`。

`fs_flags`来确认`file system`使用哪些资源来`mount`。
```C
static struct file_system_type ramfs_fs_type = {
        .name           = "ramfs",
        .mount          = ramfs_mount,
        .kill_sb        = ramfs_kill_sb,
        .fs_flags       = FS_USERNS_MOUNT,
};

static int __init init_ramfs_fs(void)
{
        if (test_and_set_bit(0, &once))
                return 0;
        return register_filesystem(&ramfs_fs_type);
}
```
`load`的方式首先是确认`mount`和相关函数的接口，然后再通过`register_filesystem`来适配起来。

之后确实涉及了其他的一些步骤，感觉学习到的还挺多。

其中还有buffer cache这个东西，感谢xv6让我前瞻了一下。