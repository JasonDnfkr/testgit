## 1 数据结构

### 1.1 文件系统 VFS 层相关数据结构

#### struct file

- `struct file` 是内存中对打开文件的描述，用于关联内存中的 `inode` 和 `path` 结构。
- **它还记录了文件的偏移** `loff_t f_pos`。
- 对于一个文件，inode对象唯一，**一个文件可以被多个进程打开，其 file 对象可以有多个。**

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

### 2.0 文件系统分层

### 2.0.1 各层介绍

- VFS 层：该层屏蔽了下层的具体操作，为上层提供统一的接口，如 vfs_read、vfs_write 等。vfs_read，vfs_write 通过调用下层实际文件系统的接口来实现相应的功能。

  > - 假设想要写个内核文件系统，那么只需要按照 Linux 预设的一些 API 接口，实现起来就行了

- 文件系统层：该层针对每一类文件系统都有相应的操作和实现，针对 VFS 层实现了各自的接口，包含了具体文件系统的处理逻辑。

- page cache层：该层缓存了从块设备中获取的数据。引入该层的目的是避免频繁的块设备访问，如果在page cache中已经缓存了I/O请求的数据，则可以将数据直接返回，无需访问块设备。

- 通用块设备层：接收上层的I/O请求，并最终发出I/O请求。该层向上层屏蔽了下层设备的特性。

- I/O调度层: 接收通用块层发出的 IO 请求，缓存请求并试图合并相邻的请求（如果这两个请求的数据在磁盘上是相邻的）。并根据设置好的调度算法，回调驱动层提供的请求处理函数，以处理具体的 IO 请求。

- SCSI 块设备驱动层：从上层取出请求，根据参数，操作具体的设备。

- 块设备：真正的物理设备。



#### 2.0.2 

### 2.1 概述

- 要读一个文件，首先通过 `read` 用户态系统调用接口，进入内核态；
- 然后执行一些函数（`ksys_read()` -> `vfs_read()`），作用是根据 fd 文件描述符取出 `struct file` 结构，并进行一些权限位的检查；
-  在 VFS 层中调用文件系统的接口函数 `f_op->read_iter()`，继而转换至一个通用通用读取函数 `generic_file_read_iter()`，其中会调用 `filemap_read()`，这个函数内部决定了是 Direct IO 还是 Buffer IO。在读写时，会先查找文件的 pagecache 映射，如果有，则直接对页面缓存进行读取操作，返回到用户空间。如果没有缓存，则从磁盘中读取数据，并映射到 pagecache 中。
- 在从磁盘中读取数据，映射到内存页的过程中，需要将读写请求整合成 bio 请求，并根据一定的调度机制，将多个读写请求进行整理后，下发至设备驱动。



### 2.2 read 详细流程

- 先走用户态系统调用接口，根据系统调用号，找到对应的内核态系统调用

- `ksys_read`：根据 fd 查找进程打开文件的实例，读取偏移。实际上，就是读取 struct file 结构

- `vfs_read`：

  - 验证权限，对文件的模式及用户 buf 进行检测，校验用户读取的区域是否有效。

  - 判断当前文件系统是否实现了对应的 read 接口。如果该 fs 有自己对应的方法，就调用，否则就调用 VFS 默认的读写方法；**其实到这里就会将磁盘数据依据用户给定的参数返回到用户空间的内存里了，不过其实际的执行流程内容很多，继续讨论；**

  - > 这里涉及到  `f_op->read()` 和 `f_op->read_iter()`，后者可以读取多个片段。**大多数文件系统都是实现了read_iter，然后会进入** `new_sync_read`

  

- `new_sync_read`：将参数 `char* buf`，`size len` 封装成两个控制读写的结构体(vec)，命名为 `struct kiocb` 和 `struct iov_iter `。

  - `struct kiocb` 封装了 `struct file *`，用于控制内核的读写流程（当前I/O的状态）。内核的每一次读写都会对应一个 `kiocb`。

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

 

- `file->f_op->read_iter(kio, iter);` -> `generic_file_read_iter`：封装完成之后，会调用一个通用的读文件接口，**在此处判断是直接读写 Direct IO 还是缓存读写 Buffer IO**。该函数有两个执行路径，如果是以O_DIRECT方式打开文件，则读操作跳过page cache，直接发动块设备IO请求，读取磁盘；否则尝试从page cache中获取所需的数据

 

- 讨论 Buffer IO 的情况：这里开始涉及到了 page cache 层，调用 `filemap_read`，该函数逻辑是，如果所需数据存在于 page cache 中，并且数据不是脏的，则从 page cache 中直接获取数据返回；如果数据在 page cache 中不存在，或者数据是脏的，则 page cache 会引发读磁盘的操作。并且会有一定的预读机制来提高 cache 的命中率，减少磁盘访问的次数。读磁盘具体来说，是：

  - 先尝试找到对应的 page，如果不存在，则创建，将文件内容读取到 page 中， 然后再拷贝到对应的 `iov` 结构里（封装了 buf 和 len 的结构体）。**这一段的机制十分复杂，涉及到预读等流程**。

  - 预读的流程，还不太了解 // TODO

    

### 2.3 内核如何从磁盘上读取数据

#### 2.3.1 总体逻辑

- 一个文件在磁盘上的分布，通过文件系统的数据结构进行有规律的组织。它可能连续分布在磁盘上的，也可能不连续。可以通过文件的 inode 号，对应到文件系统上该文件的编号，然后根据文件系统的数据结构组织，找到文件的位置。
- **块设备**每次读取都必须以 `sector` 扇区（物理上叫扇区，内核视角通常描述为块） 512B 为单位，因此可以计算出读取该文件，涉及到哪些 sector，和这些 sector 的位置。在内核中，与磁盘上的 sector 对应的数据结构是 `buffer_head`。 
  - `get_block` 函数会创建一些 `buffer_head` 结构体，通过文件的 inode 号，查找到文件具体所在的 `sector`，建立映射。
- **内核**每次读取都必须以 page 为单位，也就是 4KB。因此，一个页包括很多个 sector，因此对于 page cache 而言，每个 4KB 的 page 都会对应到磁盘上 4KB 的连续块上面，内核需要以页为单位，读取出文件涉及到的每一个 sector。因此，page 与 buffer_head 在数据结构上也相关联。
- `bio` 数据结构描述硬盘里面的位置与 page cache 的页的对应关系。bio 结构中，有一个 `bio_vec` 迭代器，指向一张表，描述一次请求中对应的 page。
- bio 将连续的页面作为一次请求提交。在内核从磁盘读取数据映射到 page cache 的过程中，会尝试做以下内容：
  - 如果磁盘上物理块连续，则只需要提交一个 bio 结构即可；
  - 如果磁盘上物理块部分连续，则将连续的部分合并成一个 bio，其余离散的部分每一页都分别构造一个 bio；
  - 如果都不连续，比如文件分散在4个离散的页里，则构造4个 bio。



#### 2.3.2 bio 提交、request

- 使用 `submit_bio` 函数，对 `bio` 进行提交操作，把 `bio` 封装成对应磁盘请求的 `request` 结构。
- 从机制上来说，内核会将多个 `request` 请求，按照一定的算法（如电梯调度算法）等添加到一个队列中。
- 当 I/O 请求到达一定限制时，内核会将多个 `request` 请求进行排序，最后向驱动层发送读请求。
- 再往下就是设备驱动层了，涉及到 scsi，ufs，不太了解



## 3 怎么刷盘

分两种情况：

- writeback 数据回刷方式，write 调用 + sync 调用；
- Direct IO 直接读写方式

对于 Buffer IO，VFS 到文件系统的读写，只是将数据从物理硬盘中取出，在内存中进行修改。需要有一个回刷机制，将缓存落盘。

触发数据回刷的方式如下：

- 经过特定时间后（30s？不确定）
- 脏页的数量超过了一定值
- 调用了 sync

内核有一个线程会自动挑选块设备对应的一些脏文件，把文件对应的 page 刷写到磁盘。落盘的执行，需要回调文件系统提供的 `write_page` 等接口。由于之前“从物理硬盘读取块，组成页，存入 page cache” 这一过程中，已经通过 buffer_head 等手段，确定了 page 对应磁盘的物理位置，因此可以直接方便地写入。

