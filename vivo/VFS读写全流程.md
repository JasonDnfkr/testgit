## 1 数据结构

### 1.1 文件系统 VFS 层相关数据结构

#### struct file

- `struct file` 是内存中对打开文件的描述，用于关联内存中的 `inode` 和 `path` 结构。
- **它还记录了文件的偏移** `loff_t f_pos`。
- 对于一个文件，inode对象唯一，**一个文件可以被多个进程打开，其file对象可以有多个。**

```c
struct file {
	fmode_t			f_mode;
	atomic_long_t		f_count;

	loff_t			f_pos;
	unsigned int		f_flags;

	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	struct address_space	*f_mapping;
};
```



##### fd 和 file 和 进程 的关系？

回答：每个进程都有一个打开文件表，内核可以直接获取当前进程的信息，调用这个表。这个表实际上是一个由 `struct file*`  构成的指针数组。文件描述符 fd 就是从这个数组里取出下标索引，作为返回值。



#### struct path

- `struct path` 结构表示文件的路径，通常是从字符串路径解析之后，在内存中的结构。

```c
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
};
```



#### struct dentry

- `struct dentry` 是目录项缓存，用于将缓存的路径和 inode 结合，**以便能够通过目录路径字符串快速找到 inode。**
- 在路径查找的过程中，每查询到一个目录，就会建立一次 dentry 到 inode 的缓存。
- dentry 以树状的结构相连：所有的 dentry 用 d_parent 和 d_child 连接起来，形成树状结构。

```c
struct dentry {
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */
}
```



#### struct inode

- 包含了内核在操作文件或目录时需要的全部元数据（文件访问权限、属主、组、大小、生成时间、访问时间、最后修改时间等信息）
- inode 结构中的静态信息取自物理设备上的文件系统，**由文件系统指定的函数填写**，它只存在于内存中，可以通过 inode 缓存访问

```c
// <linux/fs.h>
struct inode
{
    /* 相关权限位控制 */
    umode_t i_mode;
    unsigned short i_opflags;
    kuid_t i_uid;
    kgid_t i_gid;
    unsigned int i_flags;

    const struct inode_operations *i_op; // 方法集合
    struct super_block *i_sb; 		     // 自己指向的文件系统超级块
    struct address_space *i_mapping;	 // 该inode的文件内存缓存

    unsigned long i_ino;	// inode号
  	
    dev_t i_rdev;
    loff_t i_size;
    struct timespec i_atime;
    struct timespec i_mtime;
    struct timespec i_ctime;
    
    /* 用于定位磁盘块位置的信息 */
    unsigned short i_bytes;
    unsigned int i_blkbits;
    blkcnt_t i_blocks;
};
```

```c
struct inode_operations
{
    struct dentry *(*lookup)(struct inode *, struct dentry *, unsigned int);

    // 创建文件，由create和open系统调用调用
    int (*create)(struct inode *, struct dentry *, umode_t, bool);
    int (*link)(struct dentry *, struct inode *, struct dentry *);
    int (*unlink)(struct inode *, struct dentry *);
    int (*symlink)(struct inode *, struct dentry *, const char *);
    int (*mkdir)(struct inode *, struct dentry *, umode_t);
    int (*rmdir)(struct inode *, struct dentry *);
    int (*mknod)(struct inode *, struct dentry *, umode_t, dev_t);
    int (*rename)(struct inode *, struct dentry *,
                  struct inode *, struct dentry *);
    int (*setattr)(struct dentry *, struct iattr *);
    int (*getattr)(struct vfsmount *mnt, struct dentry *, struct kstat *);
    int (*setxattr)(struct dentry *, const char *, const void *, 
} ____cacheline_aligned;
```



#### struct super_block

- super_block 是文件系统的元数据集合，用于存储特定文件系统的信息
- 通常对应于存放在磁盘特定扇区中的文件系统超级块，挂载时会自动读取文件系统中对应的超级块，加载到内存



## 2 read 流程

### 2.1 概述

要读一个文件，首先通过 `read` 用户态系统调用接口，进入内核态，执行至 `ksys_read()` -> `vfs_read()`，在 VFS 层中调用文件系统的接口 `f_op->read_iter()`，继而转换至通用接口 `generic_file_read_iter()`，其中会调用 `filemap_read()`，这个函数内部决定了是 Direct IO 还是 Buffer IO。在读写时，会先查找文件的 pagecache 映射，如果有，则直接对页面缓存进行读取操作，返回到用户空间。如果没有缓存，则从磁盘中读取数据，并映射到 pagecache 中。



### 2.2 read 详细流程

- `ksys_read`：根据 fd 查找进程打开文件的实例，读取偏移



- `vfs_read`：

  - 验证权限，对文件的模式及用户 buf 进行检测，校验用户读取的区域是否有效。
  - 判断当前文件系统是否实现了对应的 read 接口。如果该 fs 有自己对应的方法，就调用，否则就调用 VFS 默认的读写方法。
  - 这里涉及到  `f_op->read()` 和 `f_op->read_iter()`，后者可以读取多个片段。**大多数文件系统都是实现了read_iter，然后会进入** `new_sync_read`

  

- `new_sync_read`：将参数 `char* buf`，`size len` 封装成两个控制读写的结构体(vec)，命名为 `struct kiocb` 和 `struct iov_iter `。

  - `struct kiocb` 封装了 `struct file *`，用于控制内核的读写流程。内核的每一次读写都会对应一个 `kiocb`。

  - `struct iov_iter` 是一个迭代器结构，封装了 buf 和 len。**这个迭代器用于记录所有需要读取的片段信息，因为内核有系统调用支持读写多个非连续缓冲区。**

  - 这两个封装的结构，简单来说，用于内核态到用户态的数据拷贝。

  - ```c
    struct iov_iter {
        size_t count;	// 需要读写的总长度
        union {
            const struct iovec *iov; // 指向一个数组，代表所有片段信息
            const struct kvec *kvec;
            const struct bio_vec *bvec;
            struct pipe_inode_info *pipe;
        };
        unsigned long nr_segs; // iov的数组长度
        };
    };
    ```

 

- `file->f_op->read_iter(kio, iter);` -> `generic_file_read_iter`：调用一个通用的读文件接口，**在此处判断是直接读写 Direct IO 还是缓存读写 Buffer IO**。

 

- 然后讨论 Buffer IO：调用 `filemap_read`，该函数逻辑是，
  - 将文件内容读取到 page 中， 然后再拷贝到对应的 `iov` 结构里（封装了 buf 和 len 的结构体）
  - 找到对应的 page，如果不存在，则新建，并将文件内容读取到其中。**这一段的机制十分复杂，涉及到预读等流程**。
  - 
  - 遍历相关的 page， 将其内容拷贝到 `iov_iter` 迭代器中，并复制到最初的 buf 地址中。



### 2.3 如何从块设备读取到内存页(bio、buffer_head)

- 文件读取（或者预读）的最后一步，是将块设备上的内容读取到 page 中。
- 调用 `address_space_operations` 的 `readpage` 函数，这个函数会调用 `do_mpage_readpage` ，**这是读取块设备内容到缓存页面的关键函数。**

#### 2.3.1 bio 和 buffer_head

- VFS 下发一个 IO 请求到 block 层，一般涉及到俩个概念：`bio` 和 `buffer_head`。
- `bio` 代表物理连续的多个 `sector`，用于请求一大段连续空间
- `buffer_head` 对应一个磁盘上的 `sector`，也就是 512B
- 一般来说，文件在磁盘里不一定是连续扇区存放的。因此，读磁盘块的函数 `do_mpage_readpage`，需要做到能够准确计算出 `buffer_head` 对应的物理 `sector`。
- 这个过程，需要回调文件系统自身的函数 `get_block`，**该函数作用是根据 VFS 的 inode，查找缓存页中的逻辑** `sector` 对应的物理 `sector`，**并建立** `buffer_head` **和** `sector` **的关系**：也就是建立地址映射。



#### 2.3.2 bio 提交、request

- `bio` 会尽可能地将 `buffer_head` 组合（存疑），并将多个 `bio` 请求，尽可能的合并到一起。
- 使用 `submit_bio` 函数，对 `bio` 进行提交操作，把 `bio` 封装成对应磁盘请求的 `request` 结构。
- 从机制上来说，内核会将多个 `request` 请求，按照一定的算法（如电梯调度算法）等添加到一个队列中。
- 当 I/O 请求到达一定限制时，内核会将多个 `request` 请求进行排序，最后向驱动层发送读请求。
- 再往下就是设备驱动层了，涉及到 scsi，ufs，不太了解





