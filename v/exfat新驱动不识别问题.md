# exfat新驱动不识别文件问题

## 一、问题背景

PD2183、PD2171等项目，无法识别exfat格式U盘里的文件



## 二、EXFAT介绍

exfat也叫fat64，是微软专门为可移动设备设计的。

![img](E:\Data\Work\Job\vivo\秋招复习\exfat新驱动不识别问题.assets\clip_image002.png)

https://docs.microsoft.com/en-us/windows/win32/fileio/exfat-specification#77-file-name-directory-entry

## 三、问题分析

测试同事反馈，PD2183项目机器无法识别U盘内容，而且先插到安卓11的机器再插入PD2183会重复创建相同文件夹。

 

经过排查，安卓12 使用5.10 kernel的机型都有此表现。

两者差异：

1.kernel 5.10采用linux原生exfat驱动，之前的项目exfat使用的驱动来自github（https://github.com/dorimanx/exfat-nofuse）。

2.kernel 5.10采用新的fsck工具（https://github.com/exfatprogs/exfatprogs），之前的项目exfat使用的fsck工具源码未知。

对比两者挂载参数也有差异，kernel 5.10驱动未支持namecase参数：

```
PD2154:/ # mount | grep exfat
/dev/block/vold/public:8,97 on /mnt/media_rw/38F0-56EF type exfat (rw,dirsync,nosuid,nodev,noexec,relatime,uid=1023,gid=1023,fmask=0007,dmask=0007,allow_utime=0020,iocharset=utf8,namecase=1,errors=remount-ro)
 
PD2183:/ # mount | grep exfat
/dev/block/vold/public:8,49 on /mnt/media_rw/38F0-56EF type exfat (rw,dirsync,nosuid,nodev,noexec,relatime,uid=1023,gid=1023,fmask=0007,dmask=0007,allow_utime=0020,iocharset=utf8,errors=remount-ro)
```

 

PD2183项目机器无法识别U盘内容，将U盘格式化后，可以正常使用U盘。

首先排除工具影响，将PD2183手机里fsck.exfat、fsck.exfat_v2工具重命名为fsck.exfat.copy、fsck.exfat_v2.copy。

工具重命名后，挂载外置存储 check时找不到fsck工具，会导致无法正常挂载。此时，可以通过命令行手动挂载。

 

连接wifi adb时，使用要求：

1.有笔记本电脑或无线网卡

2.使用额外手机创建热点，并将机器和电脑连接至该热点

具体步骤可以参考：[Wifi adb使用说明](http://km.vivo.xyz/pages/viewpage.action?pageId=34830597)

 

![image-20230708212301448](E:\Data\Work\Job\vivo\秋招复习\exfat新驱动不识别问题.assets\image-20230708212301448.png)

在去掉fsck工具后，通过命令行手动挂载U盘，在老驱动上以不同namecase挂载参数创建文件后，插入新驱动机器。经试验发现：

1.老驱动上使用namecase=0参数挂载后创建的文件，在新驱动上可以正常识别。

2.老驱动上使用namecase=1参数挂载后创建的文件，在新驱动上无法正常识别。

同时，发现经过老工具以-fR参数（正常为-cfR）修复后的U盘，在新驱动上可以正常识别。

**由此推测，该问题是由于安卓12用的kernel不支持识别namecase参数，识别文件metadata有问题，且老工具能够修复文件metadata。**

 

为了证实以上推测，可以采用两个方法来证实：

**1. 使用[winhex](https://baike.baidu.com/item/winhex/4558863?fr=aladdin)工具，对比新老驱动生成同名文件metadata差异，或者对比使用老驱动生成的文件在老工具修复前后的差异。**

![image-20230708212419522](E:\Data\Work\Job\vivo\秋招复习\exfat新驱动不识别问题.assets\image-20230708212419522.png)

![image-20230708212439535](E:\Data\Work\Job\vivo\秋招复习\exfat新驱动不识别问题.assets\image-20230708212439535.png)

左边为老驱动上使用namecase=1参数挂载后创建的文件，该文件在新驱动上无法正常识别。右边为老工具以-fR参数（正常为-cfR）修复后的文件。

对比老工具修复前后的差异，checksum和name hash有所改变，证实了之前的推特。老工具修复了文件metadata，修复后新驱动能正常识别。

**2. 通过调试kernel代码，定位到导致不识别问题的具体位置。**

![image-20230708215029050](E:\Data\Work\Job\vivo\秋招复习\exfat新驱动不识别问题.assets\image-20230708215029050.png)

新驱动无法识别U盘文件，可以看到父目录下的子文件名称，但是文件的属性（修改时间、inode号、ugo权限等）无法正常查看，访问具体某个文件时返回ENOENT (No such file or directory)。

访问某个文件属于inode operations，对文件进行读写属于file operations。访问某个文件属性报错，推测是lookup接口有异常。 

```C
//dir
const struct inode_operations exfat_dir_inode_operations = {        
    .create     = exfat_create,                                     
    .lookup     = exfat_lookup,                                     
    .unlink     = exfat_unlink,                                     
    .mkdir      = exfat_mkdir,                                      
    .rmdir      = exfat_rmdir,                                      
    .rename     = exfat_rename,                                     
    .setattr    = exfat_setattr,                                    
    .getattr    = exfat_getattr,                                    
};                                 
 
const struct file_operations exfat_dir_operations = {
    .llseek     = generic_file_llseek,
    .read       = generic_read_dir,
    .iterate    = exfat_iterate,
    .fsync      = exfat_file_fsync,
};                                 
 
//file
const struct inode_operations exfat_file_inode_operations = {
    .setattr     = exfat_setattr,
    .getattr     = exfat_getattr,
};
 
const struct file_operations exfat_file_operations = {
    .llseek     = generic_file_llseek,
    .read_iter  = generic_file_read_iter,
    .write_iter = generic_file_write_iter,
    .mmap       = generic_file_mmap,
    .fsync      = exfat_file_fsync,
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
};
```

 

使用strace工具，追踪ls命令流程,可以发现执行newfstatat系统调用时报错：

```
1|PD2183:/ # ./data/strace ls -l /mnt/media_rw/Alarms
 
 
......
newfstatat(AT_FDCWD, "/mnt/media_rw/Alarms", 0x7fde77ccd0, AT_SYMLINK_NOFOLLOW) = -1 ENOENT (No such file or directory)
write(2, "ls: ", 4ls: )                     = 4
write(2, "/mnt/media_rw/Alarms", 20/mnt/media_rw/Alarms)    = 20
write(2, ": No such file or directory", 27: No such file or directory) = 27
write(2, "\n", 1
)                       = 1
exit_group(1)                           = ?
+++ exited with 1 +++
1|PD2183:/ #
```

 

newfstatat系统调用实现在fs/stat.c中，函数传参为dfd、文件路径、stat结构体、flag。

```C
/*
 * dfd: 可以是一个目录的fd，也可以是特殊值AT_FDCWD；如果文件路径指定的是文件的绝对路径，则此参数会被忽略
 * filename: 指定文件路径。如果pathname参数指定的是一个相对路径、并且dirfd参数不等于特殊值AT_FDCWD，则实际操作的文件路径是相对于文件描述符dirfd指向的目录进行解析。
 *                      如果pathname参数指定的是一个相对路径、并且dirfd参数等于特殊值AT_FDCWD，则实际操作的文件路径是相对于调用进程的当前工作目录进行解析.
 */
SYSCALL_DEFINE4(newfstatat, int, dfd, const char __user *, filename,
        struct stat __user *, statbuf, int, flag)
{
    struct kstat stat;
    int error;
 
    error = vfs_fstatat(dfd, filename, &stat, flag);
    if (error)
        return error;
    return cp_new_stat(&stat, statbuf);
}
 
->newfstatat
        ->vfs_fstatat
               ->vfs_statx
                       ->user_path_at
                               ->user_path_at_empty
                                      ->filename_lookup
                                              ->path_lookupat
                                                      ->link_path_walk
                                                             ->walk_component
                                                                     ->lookup_slow
                                                                             ->__lookup_slow
                                                                                    ->exfat_lookup
                       ->vfs_getattr
                               ->security_inode_getattr
                                      ->selinux_inode_getattr
                                      ->vfs_getattr_nosec
                                              ->exfat_getattr
 
 
```

newfstatat主要完成两个工作，执行具体文件系统lookup接口，查找到文件缓存起来后，通过具体文件系统getattr接口获取文件属性。

exfat当前不支持trace event，可以通过ftrace中kretprobe和kprobe对文件lookup流程进行追踪。

经过追踪，定位到lookup中报错位置，新驱动在exfat_find_dir_entry函数中遍历exfat entry时报错。

 展开源码

```C
struct exfat_dentry {
    __u8 type;
    union {
        struct {
            __u8 num_ext;
            __le16 checksum;
            __le16 attr;
            __le16 reserved1;
            __le16 create_time;
            __le16 create_date;
            __le16 modify_time;
            __le16 modify_date;
            __le16 access_time;
            __le16 access_date;
            __u8 create_time_cs;
            __u8 modify_time_cs;
            __u8 create_tz;
            __u8 modify_tz;
            __u8 access_tz;
            __u8 reserved2[7];
        } __packed file; /* file directory entry */
        struct {
            __u8 flags;  TER
            __u8 reserved1;
            __u8 name_len;
            __le16 name_hash;
            __le16 reserved2;
            __le64 valid_size;
            __le32 reserved3;
            __le32 start_clu;
            __le64 size;
        } __packed stream; /* stream extension directory entry */
        struct {
            __u8 flags;
            __le16 unicode_0_14[EXFAT_FILE_NAME_LEN];
        } __packed name; /* file name directory entry */
        struct {
            __u8 flags;
            __u8 reserved[18];
            __le32 start_clu;
            __le64 size;
        } __packed bitmap; /* allocation bitmap directory entry */
        struct {
            __u8 reserved1[3];
            __le32 checksum;
            __u8 reserved2[12];
            __le32 start_clu;
            __le64 size;
        } __packed upcase; /* up-case table directory entry */
    } __packed dentry;
} __packed;
->exfat_lookup
        ->exfat_find_dir_entry
 
fs/exfat/dir.c
exfat_find_dir_entry:
 
            if (entry_type == TYPE_UNUSED ||
                entry_type == TYPE_DELETED) {
                step = DIRENT_STEP_FILE;
 
                num_empty++;
                if (candi_empty.eidx == EXFAT_HINT_NONE &&
                        num_empty == 1) {
                    exfat_chain_set(&candi_empty.cur,
                        clu.dir, clu.size, clu.flags);
                }
 
                if (candi_empty.eidx == EXFAT_HINT_NONE &&
                        num_empty >= num_entries) {
                    candi_empty.eidx =
                        dentry - (num_empty - 1);
                    WARN_ON(candi_empty.eidx < 0);
                    candi_empty.count = num_empty;
 
                    if (ei->hint_femp.eidx ==
                            EXFAT_HINT_NONE ||
                        candi_empty.eidx <=
                             ei->hint_femp.eidx)
                        ei->hint_femp = candi_empty;
                }
 
                brelse(bh);
                if (entry_type == TYPE_UNUSED)
                    goto not_found;                                                 <-进入异常分支
                continue;
            }
```

对比正常识别流程，在遍历exfat entry时，经过TYPE_DIR、TYPE_STREAM、TYPE_EXTEND三种遍历后走到found分支。

```C
fs/exfat/dir.c
exfat_find_dir_entry:
 
            if (entry_type == TYPE_EXTEND) {
                unsigned short entry_uniname[16], unichar;
 
                if (step != DIRENT_STEP_NAME) {
                    step = DIRENT_STEP_FILE;
                    continue;
                }
 
                if (++order == 2)
                    uniname = p_uniname->name;
                else
                    uniname += EXFAT_FILE_NAME_LEN;
 
                len = exfat_extract_uni_name(ep, entry_uniname);
                name_len += len;
 
                unichar = *(uniname+len);
                *(uniname+len) = 0x0;
 
                if (exfat_uniname_ncmp(sb, uniname,
                    entry_uniname, len)) {
                    step = DIRENT_STEP_FILE;
                } else if (p_uniname->name_len == name_len) {
                    if (order == num_ext)
                        goto found;                                                                 <-正常流程
                    step = DIRENT_STEP_SECD;
                }
 
                *(uniname+len) = unichar;
                continue;
            }
```

经过debug，发现异常时，匹配TYPE_STREAM时，newfstatat路径计算的name_hash与exfat stream entry中name_hash比较失败。

```C
fs/exfat/dir.c
exfat_find_dir_entry:
 
            if (entry_type == TYPE_STREAM) {
                u16 name_hash;
 
                if (step != DIRENT_STEP_STRM) {
                    step = DIRENT_STEP_FILE;
                    brelse(bh);
                    continue;
                }
                step = DIRENT_STEP_FILE;
                name_hash = le16_to_cpu(
                        ep->dentry.stream.name_hash);
                if (p_uniname->name_hash == name_hash &&     <-未匹配成功
                    p_uniname->name_len ==
                        ep->dentry.stream.name_len) {
                    step = DIRENT_STEP_NAME;
                    order = 1;
                    name_len = 0;
                }
                brelse(bh);
                continue;
            }
```

 

阅读代码可知，name_hash 是文件、目录在重命名、移动或创建操作时，exfat_resolve_path函数解析路径后生成，并在exfat_init_ext_entry函数中写回磁盘。问题排查到这里比较清晰，就是由于老驱动生成文件(使用namecase=1挂载参数)的namehash值计算方式不同导致。

```C
->exfat_rename
        ->exfat_resolve_path
               (1)
               ->exfat_rename_file
                       ->exfat_init_ext_entry
               (2)
               ->exfat_move_file
                       ->exfat_init_ext_entry
 
 
->exfat_create
        ->exfat_resolve_path
        ->exfat_add_entry
               ->exfat_init_ext_entry
 
->exfat_mkdir
        ->exfat_resolve_path
        ->exfat_add_entry
               ->exfat_init_ext_entry
```

同时老驱动resolve_path函数中，计算hash时会根据namecase大小写参数对字符进行处理，返回原字符或查表后的值。

```C
void nls_cstring_to_uniname(struct super_block *sb, UNI_NAME_T *p_uniname, u8 *p_cstring, s32 *p_lossy)
{
    int i, j, lossy = FALSE;
    u8 *end_of_name;
    u8 upname[MAX_NAME_LENGTH * 2];
    u16 *uniname = p_uniname->name;
    struct nls_table *nls = EXFAT_SB(sb)->nls_io;
                                        
    /* strip all trailing spaces */
    end_of_name = p_cstring + strlen((char *) p_cstring);
 
    while (*(--end_of_name) == ' ') {
        if (end_of_name < p_cstring)
            break;
    }
    *(++end_of_name) = '\0';
 
    if (strcmp((char *) p_cstring, ".") && strcmp((char *) p_cstring, "..")) {
 
        /* strip all trailing periods */
        while (*(--end_of_name) == '.') {
            if (end_of_name < p_cstring)
                break;
        }
        *(++end_of_name) = '\0';
    }
 
    if (*p_cstring == '\0')
        lossy = TRUE;
 
    if (nls == NULL) {
        i = utf8s_to_utf16s(p_cstring, MAX_NAME_LENGTH, UTF16_HOST_ENDIAN, uniname, MAX_NAME_LENGTH);
        for (j = 0; j < i; j++)
            SET16_A(upname + j * 2, nls_upper(sb, uniname[j]));                                            <-此处
        uniname[i] = '\0';
    }
    else {
        i = j = 0;
        while (j < (MAX_NAME_LENGTH-1)) {
            if (*(p_cstring+i) == '\0')
                break;
 
            i += convert_ch_to_uni(nls, uniname, (u8 *)(p_cstring+i), &lossy);
 
            if ((*uniname < 0x0020) || nls_wstrchr(bad_uni_chars, *uniname))
                lossy = TRUE;
 
            SET16_A(upname + j * 2, nls_upper(sb, *uniname));
 
            uniname++;
            j++;
        }
 
        if (*(p_cstring+i) != '\0')
            lossy = TRUE;
        *uniname = (u16) '\0';
    }
 
    p_uniname->name_len = j;
    p_uniname->name_hash = calc_checksum_2byte((void *) upname, j<<1, 0, CS_DEFAULT);
 
    if (p_lossy != NULL)
        *p_lossy = lossy;
} /* end of nls_cstring_to_uniname */
```

## 四、问题解决

明确是U盘中文件name hash与新驱动计算出的结果不匹配后，有3种修复方式：

  1.exfat新驱动 exfat_find_dir_entry函数中检查stearm entry时，跳过对name hash的判断。

  2.fsck新工具支持解析namecase参数并修复文件name hash。

  3.exfat新驱动 支持解析namecase参数，根据参数name hash采用不同计算方式。 

 

方式1，不需要在挂载前增加额外修复耗时，需要改动kernel，在内销项目上适用，临时规避，http://smartgit/gerrit/#/c/5298329/。

方式2，不改动kernel，在挂载前对U盘文件进行修复，可以在内外销项目上使用，http://smartgit/gerrit/#/c/5307366/。

方式3，需要改动kernel，且不符合exfat规范（https://docs.microsoft.com/en-us/windows/win32/fileio/exfat-specification#77-file-name-directory-entry）。

All children of a given directory entry shall have unique File Name Directory Entry Sets.

That is to say there can be no duplicate file or directory names after up-casing within any one directory.