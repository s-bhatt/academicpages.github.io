---
title: "Designing an Operating System - III"
collection: projects
type: "Project"
permalink: /projects/2018-01-01-operating-system-3
venue: "Computer Science Department, Rutgers University"
date: 2018-01-01
location: "New Brunswick, New Jersey"
---

We (I along with two other super-talented blokes) designed an Operating System in three phases across the Fall of 2017. The three phases were:
* Threads and Scheduling
* Pseudo-virtual memory
* Simple File System

This writeup details out our third phase of implementation.

## Simple File System

The mapping of these various FUSE functions onto our code functions is mentioned as “struct fuse_operations vrs_oper” in sfs.c.

0. int create(char* path)
1. int delete(char* path)
2. int stat(char* path, struct stat* buf)
3. int open(char* path)
4. int close(int fileID)
5. int read(int fileID, void* buffer, int bytes)
6. int write(int fileID, void* buffer, int bytes)
7. struct dirent* readdir(int directoryID)

### Superblock

The sfs.c contains a superblock named vrs_superblock. This super block contains the data_blocks stored in “bitmap_data_blocks”, inode blocks stored in “bitmap_inode_blocks” and the inode root directory. “num_data_blocks” stores the total number of data blocks on disk. Similarly, “num_free_blocks” maintains the total number of free blocks and “num_inodes” corresponds to the total number of inodes on disk.

	typedef struct __attribute__((packed)) {
		uint32_t magic;
		uint32_t num_data_blocks;
		uint32_t num_free_blocks; 
		uint32_t num_inodes; 
		uint32_t bitmap_inode_blocks;
		uint32_t bitmap_data_blocks;
		uint32_t inode_root;  
	} vrs_superblock;





### File System Calls

The void *vrs_init(struct fuse_conn_info *conn) establishes connection . Further, in init, we initialize “vrs_superblock sb” using our declarations from inode.h and write it to disk file.
We then write the “inode bitmap” using “block_write” defined in block.c. We use a similar process for data bitmap, inode blocks, data blocks and the root inode.

Next we cache the state of inode’s and data block's availability in fuse context.

The vrs_open(const char *path, struct fuse_file_info *fi) opens a file by calling “path_2_ino” using the user specified path and then fetches the inode from that path.

The vrs_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) fetches the inode using the path from “path_2_ino()”. It then calls “read_inode(&inode, buf, size, offset)” using the inode and user specified buffer as parameters.

vrs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) has a similar implementation as vrs_read, with the exception that “write_inode(&inode, buf, size, offset)” is called instead.

int vrs_release(const char *path, struct fuse_file_info *fi) implements file close function.

The vrs_destroy function simply sets the state variables to NULL;


### Inode

Moving onto inode.h, we have made all the declarations of the vrs_superblock over here:

	VRS_BLOCK_SUPERBLOCK is set to 0.
	VRS_BLOCK_INODE_BITMAP is set to 1 since there is only 1 super block.
	VRS_BLOCK_DATA_BITMAP hence becomes 2.
	VRS_BLOCK_INODES is set to 1026.
	and finally VRS_BLOCK_DATA is set to 1026 + 64.

The vrs_inode_t struct has the following attributes:


* ino: inode number
* mode: Flags related to file mode 
* nlink: number of hard links
* size: total size, in bytes
* nblocks: number of 512B blocks allocated
* atime: time of last access
* mtime: time of last modification
* ctime: time of last status change
* blocks[VRS_N_BLOCKS]: Size. This has been set to 4*15 = 60 bytes

