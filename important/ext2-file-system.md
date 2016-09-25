---
title: Ext2文件系统简述
date: 2016-09-23 17:57:48
tags:
- Ext2 
- 文件系统
categories: 文件系统
---
#### Ext2磁盘数据结构
任何Ext2分区中的第一个块从不受Ext2文件系统的管理，因为第一个块是分区的引导扇区。分区中剩下来的其他部分分成了块组，其分布图如下:
![Ext2块组分布图](/images/ext2/ext2-file-system-0.png)
典型的块大小是1024 bytes或者4096 bytes。这个大小在创建Ext2文件系统的时候被决定，它可以由系统管理员指定，也可以由文件系统的创建程序根据硬盘分区的大小，自动选择一个较合理的值。

每个块组的开头部分都由超级块描述符和块组描述符占用，而且每个块组的超级块描述符和块组描述符都是相同的，冗余的主要目的就是保证超级块描述符和组描述符能在描述符数据损坏的情况下依旧能回到一个一致的状态。这里的块组描述符其实是一个块组描述符组，这n个块中存放着分区中所有的块组对应的块组描述符。只有块组0中所包含的超级块和块组描述符才由内核使用，其余的保持不变。

在Ext2中定义了如下几种数据类型:

- `__u8` 、`__u16` 、`__u32`分表表示长度为8、16、32的无符号数。
- `__s8` 、`__s16` 、`__s32`分别表示长度为8、16、32的有符号数。
- `__le16` 、`__le32`分别表示字、双字的little-endian排序方式。 
- `__be16` 、`__be32`分别表示字、双字的big-endian排序方式。

#### 超级块

超级块描述符如下：

```c
struct ext2_super_block {
        __u32 s_inodes_count;      		/* 索引节点总数 */
        __u32 s_blocks_count;      		/* 块的数量*/
        __u32 s_r_blocks_count;    		/* 保留的块数 */
        __u32 s_free_blocks_count; 		/* 空闲的块数 */
        __u32 s_free_inodes_count;      /* 空闲的索引节点数 */
        __u32 s_first_data_block;       /* 第一个使用的块号 */
        __u32 s_log_block_size;         /* 块的大小 */
        __le32  s_log_frag_size;        /* Fragment size */
        __le32  s_blocks_per_group;     /* 每个块组的块数量 */
        __le32  s_frags_per_group;      /* Fragments per group */
        __le32  s_inodes_per_group;     /* 每个块组的索引节点数量 */
        __le32  s_mtime;                /* Mount time */
        __le32  s_wtime;                /* Write time */
        __le16  s_mnt_count;            /* Mount count */
        __le16  s_max_mnt_count;        /* Maximal mount count */
        __le16  s_magic;                /* Magic signature */
        __le16  s_state;                /* File system state */
        __le16  s_errors;               /* Behaviour when detecting errors */
        __le16  s_minor_rev_level;      /* minor revision level */
        __le32  s_lastcheck;            /* time of last check */
        __le32  s_checkinterval;        /* max. time between checks */
        __le32  s_creator_os;           /* OS */
        __le32  s_rev_level;            /* Revision level */
        __le16  s_def_resuid;           /* Default uid for reserved blocks */
        __le16  s_def_resgid;           /* Default gid for reserved blocks */
        __le32  s_first_ino;            /* First non-reserved inode */
        __le16   s_inode_size;          /* size of inode structure */
        __le16  s_block_group_nr;       /* block group # of this superblock */
        __le32  s_feature_compat;       /* compatible feature set */
        __le32  s_feature_incompat;     /* incompatible feature set */
        __le32  s_feature_ro_compat;    /* readonly-compatible feature set */
        __u8    s_uuid[16];             /* 128-bit uuid for volume */
        char    s_volume_name[16];      /* volume name */
        char    s_last_mounted[64];     /* directory where last mounted */
        __le32  s_algorithm_usage_bitmap; /* For compression */
        /*
         * Performance hints.  Directory preallocation should only
         * happen if the EXT2_COMPAT_PREALLOC flag is on.
         */
        __u8    s_prealloc_blocks;      /* Nr of blocks to try to preallocate*/
        __u8    s_prealloc_dir_blocks;  /* Nr to preallocate for dirs */
        __u16   s_padding1;
        /*
         * Journaling support valid if EXT3_FEATURE_COMPAT_HAS_JOURNAL set.
         */
        __u8    s_journal_uuid[16];     /* uuid of journal superblock */
        __u32   s_journal_inum;         /* inode number of journal file */
        __u32   s_journal_dev;          /* device number of journal file */
        __u32   s_last_orphan;          /* start of list of inodes to delete */
        __u32   s_hash_seed[4];         /* HTREE hash seed */
        __u8    s_def_hash_version;     /* Default hash version to use */
        __u8    s_reserved_char_pad;
        __u16   s_reserved_word_pad;
        __le32  s_default_mount_opts;
        __le32  s_first_meta_bg;        /* First metablock block group */
        __u32   s_reserved[190];        /* Padding to the end of the block */
};
```

`s_magic`字段是 magic 签名，对于 ext2 和 ext3 文件系统来说，这个字段的值应该正好等于 0xEF53。从这里可以看出ext2 和 ext3 的兼容性一定是很强的，不然不会使用同一个magic签名。

`s_log_block_size`这个字段以2的幂次方表示的 block 的大小。我们把 block 的大小记作 B，B = 1 << (s_log_block_size + 10)，单位是 bytes。举例来说，如果这个字段是 0，那么 block 的大小就是 1024 bytes，这正好就是最小的 block 大小；如果这个字段是 1，那么 block 大小就是 2048 bytes。

`s_mnt_count `、`s_max_mnt_count `、`s_lastcheck `、`s_checkinterval `字段用于系统启动时自动检查Ext2文件系统。

`s_state`字段表示Ext2文件系统的安装和卸载情况，如果Ext2文件系统被安装或未被全部卸载，则字段的值为0；如果被正常卸载，则字段的值为1；如果包含错误，则字段的值为2。

其他常用字段的含义通过注释基本上可以理解，或者可以查阅`UTLK`。

#### 组描述符

Ext2分区中存在多少个块组，从超级块所在块的下个块开始，就跟着由相应数量组成的块组描述符组。
每一个块组描述符如下：

```c
struct ext2_group_desc
{
		__u32 bg_block_bitmap;      /* 块位图的块号 */
		__u32 bg_inode_bitmap;      /* 索引节点位图的块号 */
		__u32 bg_inode_table;       /* 索引节点表的第一个块的块号 */
 		__u16 bg_free_blocks_count; /* 空闲块的数量 */
 		__u16 bg_free_inodes_count; /* 空闲索引节点的数量 */
		__u16   bg_used_dirs_count; /* 组中目录的数量 */
		__u16   bg_pad;
		__u32   bg_reserved[3];
};
```

#### 位图

数据块位图和索引节点位图的作用主要就是记录数据块和索引节点的使用情况。0表示空闲，1表示使用中。每个位图都存放在一个单独的块中，块的大小通常为1024 bytes或者4096 bytes,所以一个单独的位图描述8192或者32768个数据块或索引节点的使用情况。当分配或释放数据块、索引节点时都要修改对应的位图。

#### 索引节点表

索引节点表由连续的块组成，所有索引节点的大小相同，都为128 bytes。索引节点的结构为`ext2_innode`:

```c
struct ext2_inode {
        __le16  i_mode;         /* File mode */
        __le16  i_uid;          /* Low 16 bits of Owner Uid */
        __le32  i_size;         /* Size in bytes */
        __le32  i_atime;        /* Access time */
        __le32  i_ctime;        /* Creation time */
        __le32  i_mtime;        /* Modification time */
        __le32  i_dtime;        /* Deletion Time */
        __le16  i_gid;          /* Low 16 bits of Group Id */
        __le16  i_links_count;  /* Links count */
        __le32  i_blocks;       /* Blocks count */
        __le32  i_flags;        /* File flags */
        union {
                struct {
                        __le32  l_i_reserved1;
                } linux1;
                struct {
                        __le32  h_i_translator;
                } hurd1;
                struct {
                        __le32  m_i_reserved1;
                } masix1;
        } osd1;                         /* OS dependent 1 */
        __le32  i_block[EXT2_N_BLOCKS];/* Pointers to blocks */
        __le32  i_generation;   /* File version (for NFS) */
        __le32  i_file_acl;     /* File ACL */
        __le32  i_dir_acl;      /* Directory ACL */
        __le32  i_faddr;        /* Fragment address */
        union {
                struct {
                        __u8    l_i_frag;       /* Fragment number */
                        __u8    l_i_fsize;      /* Fragment size */
                        __u16   i_pad1;
                        __le16  l_i_uid_high;   /* these 2 fields    */
                        __le16  l_i_gid_high;   /* were reserved2[0] */
                        __u32   l_i_reserved2;
                } linux2;
                struct {
                        __u8    h_i_frag;       /* Fragment number */
                        __le16  h_i_mode_high;
                        __le32  h_i_author;
                } hurd2;
                struct {
                        __u8    m_i_frag;       /* Fragment number */
                        __u8    m_i_fsize;      /* Fragment size */
                        __u16   m_pad1;
                        __u32   m_i_reserved2[2];
                } masix2;
        } osd2;                         /* OS dependent 2 */
};
```

`i_size`字段表示文件的长度（字节），`i_blocks`字段存放文件占用的数据块数量（文件的数据块以512字节为一个单位）。`i_block`是一个长度为`EXT2_N_BLOCKS `的数组，每个元素都是一个指针，指向实际的文件块号，`EXT2_N_BLOCKS `通常为15，具体在后文分析。

VFS模型要求每个文件都有不同的索引节点号，但是在结构中并没有看到索引节点号。这是因为每个块组的索引节点数量相同，完全可以根据块组号、每块组索引节点数量、索引节点表来进行索引节点和块号的转换。假设每个块组包含4096个索引节点，则索引节点13021属于第三个块组，并且存放在对应索引节点表的第733项。

#### 目录
目录和普通文件一样都有着一个对应的索引节点，但是普通文件的数据块内容就是文件内容，而目录的数据块里面存放着一种数据结构`ext2_dir_entry_2`,该结构表示目录中的某一项内容，类型如下:

```c
struct ext2_dir_entry {
        __le32  inode;                  /* Inode number */
        __le16  rec_len;                /* Directory entry length */
        __le16  name_len;               /* Name length */
        __u8    file_type;		/* 文件类型 */
        char    name[];                 /* File name, up to EXT2_NAME_LEN */
};
```
`rec_len`表示目录项的长度，也可以理解为下一个目录项距当前目录项起始地址的偏移量。因为效率的原因，目录项的长度总是为4的倍数，在必要时用null字符（\0）填充文件名的末尾部分。当我们想删除一个目录项时，只需把那一项的`inode`设置为0，并且把删除项的上一项的`rec_len`加上删除项的`rec_len`(相当于把删除项给跳过)。

`name_len`表示目录项名字的实际长度。

`file_type`表示该项的文件类型，Ext2中的文件类型有如下几种:

| 文件类型| 说明|
| ----- |:------:|
| 0 | 未知 |
| 1 | 普通文件|
| 2 | 目录|
| 3 | 字符设备|
| 4 | 块设备|
| 5 | 命名管道|
| 6 | 套接字|
| 7 | 符号链接|

`name`是一个char数组，数组长度上限为`EXT2_NAME_LEN `(通常为255)。

下图是一个Ext2目录的例子:
![Ext2目录](/images/ext2/ext2-file-system-1.png)
其中文件名为`oldfile`那一项的`inode`值为0，所以该目录项已经被删除，也可以看到该项的上一项(文件名`user`)的`rec_len`为28（user项的长度加上oldfile项长度）。

#### 数据块寻址
每个非空的普通文件都含有一组数据块，这些数据块可以由所在文件的文件块号或者磁盘分区的逻辑块号来标识。在索引节点中有一个`i_block`字段，这是一个数组，元素内容就是一个指向数据块的指针，`i_block`的示意图如下:
![数据块寻址](/images/ext2/ext2-file-system-2.png)
`i_block`数组就相当于是一个目录，用于进行文件块号和逻辑块号的转换。其中0~11号元素直接进行一对一映射，将一个文件块号对应到块组数据块中的某一个逻辑快号，而下标12、13、14的元素分别指向一个一级数组、二级数组、三级数组。

从文件内的偏移量ƒ到处相应数据块的逻辑块号需要两个步骤：

1. 将偏移量ƒ转换为文件块号。
2. 将文件块号转换为数据块的逻辑块号。

假设数据块的大小为1024 bytes，如果ƒ小于1024,则ƒ对应的字符存在于第0个文件块号，也就是`i_block`数组的第0项，`i_block[0]`的内容就是该文件块号所对应的数据块块号。如果转换出来的文件块号n满足  `12 <= n <= b/4 + 11`(b表示块大小，在这里就是1024)，则表示该文件块号存在于`i_block[11]`指向的数组中。通过`i_block[11]`的值，找到一个间接块，然后找到下标为`n-12`的项，其中的值便是对应的逻辑块号。`i_block[12]`、`i_block[13]`情况类似，只不过中间多了些转换过程。

参考：

- UTLK


><font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font> 
