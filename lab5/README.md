# Lab 5 notes
lab link: https://pdos.csail.mit.edu/6.828/2017/labs/lab5/  
io protection model: https://pdos.csail.mit.edu/6.828/2016/readings/i386/s08_03.htm  
io addresses from bochs: http://bochs.sourceforge.net/techspec/PORTS.LST   
more about PIO ATA: https://wiki.osdev.org/ATA_PIO_Mode  

## JOS's FS explanation
JOS has a file system "server" that handles all the file system operations, including writing directly to disk. The disk itself writes and reads in units of 512 bytes called Sectors. The File system, however, uses a bigger unit which is called a Block (the size of the block must be  divisible by the size of the sector). The block size was chosen to be the same size as a page (4096 bytes).  
The reason why the block size is the same as the page size is not a coincidence, because we'll be using x86's paging hardware to implement a block cache.

### writing and reading blocks
The entire disk (assuming the disk is less than 3GB in size) is lazily mapped into memory starting at address `DISKMAP` in the FS server. Lazily mapped means that the actual blocks will be loaded into memory from disk only when we attempt to read them (which is done using the `void* diskaddr(uint32_t blockno)` funtion). If, for example, we want to access block 3, then we'd read the page at address `DISKMAP + 3*PGSIZE` (this works because the size of each page is exactly the size of each block). If our read results in a page fault (which occurs when we read a block for the first time), FS' page fault handler will load 4096 bytes from the disk into a page frame and map it into the corresponding page in the FS server's address space. Subsequent block reads will therefore be read from memory directly, thus functioning like a block cache.  

Block writes will be handled in a similar way: we'll be writing directly to the memory mapped above `DISKMAP`, and then, we'll presist the data using `void flush_block(void *addr)`, which will write it to the disk.

### File system format on disk
sector 0 is reserved and is not touched by JOS's FS server.  
Sector 1 stores the super block. There is nothing super in the super block actually. it's just a block that stores the metadata about the file system including: 1) magic number (so we can identify the type of the file system) 2) the number of blocks on the disk 3) the metadata of the root directory-file.
Sector 2 and up stores bitmap of free blocks (where 1 means free block and 0 means occupied block).   
The rest of the sectors contain the other file/directory data.

### How is file/directory data represented
File metadata is represted by `struct File` and is exactly 256 bytes (half a sector or 1/16 of a block). File metadata contains the file name, size, type (dir or regular file), array of 10 block numbers where the actual data is stored, and another block number of a block that stores another 4096/4 = 1024 block numbers. This means that the maximum file size is `10*4096 + 1024*4096 = 4,235,264 bytes` and the maximum overhead of storing the metadata is `256+4096 = 4,352 bytes`.  

The contents of each directory file is just the file metadata (`struct File`) of the files that this directory stores (and also a directory file must be a multiple of block size). This means that a directory can store up to `4,235,264/256 = 16,544 files` (another way to calculate it is: `(10+1024)*4096/256`).

### How the user reads and writes to files
In order to read and write files, the user space environments need to send an IPC to the file server for the various file operations (such as read, write, etc). However, in order to support more than just a file system (but also pipes and "device files"), user space programs use the file descriptor abstraction layer (located in `lib/fd.c`). A "file descriptor" represents a "handle" on an open file, and is implemented by `struct Fd`. Because we'd like to share the data of `strcut Fd` between the user environments and the file server, Each `struct Fd` is located on its own page. It is also worth mentioning that each environment has a limit on how many file descriptors it can have open (`MAXFD`).  

#### opening a file
In order to start reading from or writing to a file, we first need to "open" the file. Opening the file means that we obtain a file descriptor, and then later operations (such as reading and writing) are performed on that file descriptor. the `int open(const char *path, int mode)` function is available in `lib/file.c`, and this is one of the only few public functions exported by this file (the rest of them are private functions).  
Opening a file allocates a slot file descriptor in the user environment, and then sends an IPC request to the file server to open the file. The file server itself has a limit on how many files it can have open `MAXOPEN` in the system. If that limit is not exceeded, then the file server traverses the file system (reading from the block cache or from the disk directly), allocates a page for the file descriptor, and sends the `struct Fd` page that the user environment will map into the file descritor allocated slot. 

#### performing reads and writes
The functions read and write accept a file descriptor, but they also need to know which device to actually read from or to write to. That's why each file descriptor (`struct Fd`) contains information about which device it refers to. For example, when we used the "open" call to talk to the File Service so it can open a file, the File Service was the one that allocated the `struct Fd` and also set its device to "dev_file". This is how when we read from or write to a file using the `read` or `write` functions, the functions know which "device" to dispatch to (in the File service case, the function that gets dispatched for reading is `devfile_read`).

Here is an illustration of the control flow from the lab manual:
```
      Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

## The JOS shell
After the kernel finishes initializing various subsystems (such as the memory subsystem and the multiprocessor subsystem), and various kernel data strctures (such as user environment management), it loads the code of, and creates an environment for the File Server, whose executable is embedded in the kernel image itself. Before running the scheduler and transferring to use space, the kernel also does a similar initialization and creation of an environment called `icode`, whose responsibility will evventually be to call `init`.  
By the time icode starts, the File Server is already available, which makes it possible for icode to use the file system (notice that this is in contrast to the initialization function `i386_init` in the kernel itself, which loads the `icode` and `fs` executables from RAM and doesn't do any disk access). `icode` opens the `/motd` and prints it out to the screen. Notice that at this point, `icode` is not using file descriptor 1 (stdout) for printing because the console has not yet been configured to be associated with fd 0 (for input) and fd 1 (for output). The last thing `icode` is doing before exiting is to spawn `init`.  

## Exercise 1
_Modify env\_create in env.c, so that it gives the file system environment I/O privilege, but never gives that privilege to any other environment._  
```C
void
env_create(uint8_t *binary, enum EnvType type)
{
        // LAB 3: Your code here.
        struct Env *e = NULL;
        if (env_alloc(&e, 0) < 0) {
                panic("failed to allocate env for first env 0");
        }
        load_icode(e, binary);
        e->env_type = type;

        // If this is the file server (type == ENV_TYPE_FS) give it I/O privileges.
        // LAB 5: Your code here.
        if (type == ENV_TYPE_FS) {
                e->env_tf.tf_eflags |= FL_IOPL_3;
        }
}
```
_Do you have to do anything else to ensure that this I/O privilege setting is saved and restored properly when you subsequently switch from one environment to another? Why?_  

No, this is because the  I/O privilege setting is part of the eflags register, which is saved seperately for each enviornment. So if, for example, we switch from an i/o privilaged environment to a non i/o provilaged one, the eflags register will get reloaded and the new, non-privilaged environment won't be able to access the i/o devices.

## _see answers to the other exercises in the src_

### Memory map
```
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 */
```
