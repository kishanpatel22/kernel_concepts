# File system in xv6

* xv6 has implemented **journalising file system**, which provides variety of
  system calls like **read, write, open, close, dup, etc**.

## File system calls

* Every **proc** struct for a paritcular process contains an array of file
  descripters, **open** sys call returns the first empty index from the array
  and helps in locating the exact location of inode on disk.

* **offset** is maintained with file decripter to support sequential read/write. 

* **dup** like system call duplicate pointers in file descripter array.

* **read** and **write** system call locate the data of the file disk using inode. 

* We need lower layer functions for read and write to actaully deal with disk,
  example a **disk driver**.

* **buffer cache** cache to cache the read, write I/O sequentially.


## xv6 file handling code

* xv6 file handling code is **modular** and work is **divided in layers**.

| Layer Name | Significance | File name : functions |
|------------|--------------|-----------------------|
| System call | API to xv6 file system service |  open, read, write, close, link, pipe, mknod, unlink, fstat, mkdir, chdir, dup |
| File descripter   | deal with ofile array and access struct file  | **file.c**  : fileinit, filealloc, filedup, fileclose, file.stat, fileread, file write |
| Pathname          | locate a file on actual disk using pathname | **fs.c**  : namex, namei, nameiparent, skipelem |
| Directory         | does directory lookup or search   | **fs.c**  : dirlookup, dirlink |
| Inode             |     | **fs.c**  : init, ialloc, iupdate, iget, idup, ilock, iunlock, iput, iunlock, iput, iunlockput, itrunc, stati, readi, writei, bmap |
| Logging           |     | (Block allocation on disk : balloc, bfree) **log.c**  : begin\_op, end\_op, initlog, commit |
| Buffer Cache      |     | **bio.c**  : binit, bget, bread, bwrite, bresle |
| Disk              | all driver functions for interacting with physical disk  | **ide.c**  : idewait, ideinit, idestart, ideintr, inderw |


* **Any upper layer can call any lower layer function**.


## System call layer 

* Remember all the system calls are defined in the **user.h** file and actual 
  implementation is in **sysfile.c** and **file.c**. The system calls create 
  interrupt as defined by the macro in **usys.S** assembly code, by which the 
  control is transefered to kernel's trap function in **trap.c**, now 
  trap function in turn calls the actaul function which does the required job
  using the **sys call function pointer array**.

* The sequence of calls to actaul kernel code look like 

    ```text
               open(...)       ------>   int instruction 64 (eax = open syscall)  -------->  vector.S  ------>  alltraps
        (user process system call)             (software interrupt)                 icall                jmp        |  
                                                                                                                    |  call
                open(...)     <-------   sys_open()  <----------  syscall()    <-----------   trap <---------------+
           kernel system        call                    calls    call from array      calls                call
         call implementation                                    of function pointers 
    ```

* Note : after the int instruction the execution context happens in the kernel's
  stack, however the copying of contents needs to taking place from user stack 
  to kernel's memory. Thus for system call the kernel needs to read the memory 
  of the user process stack.

* The xv6 **syscall.c** provides a set of functions which help kernel in 
  reading the contents from the stackframe, example - **argint**, **argstr**,
  **argptr**, etc.


### sys\_open function

* The kernel implementation for open function looks like 

    ```c
        /* kernel's implementation */
        int sys_open(void);                 

        /* system call API for the user */
        int open(char *filename, int flags);
    ```

* The first thing that the kernel code for open system call does is to **read 
  the values from the user process stack** which were passed as paramters for 
  system call.
    
    ```c
        char *path;
        int omode;
        if(argstr(0, &path) < 0 || argint(1, &omode) < 0) {
            return -1;
        }
    ```

* Now there is calls to begin\_op and end\_op, which actaully mean to begin 
  transaction releated to the file system. It's the kernel programmers 
  responsibility to place begin and end correctly.
    
    ```c
        begin_op();
    ```

* The flags passed as arguments to the system call are checked if it is to
  create a new file in the file system, this is done by calling same level layer 
  **create** function which returns **inode fo the new file created** on system,
  for the **given pathname**
    
    ```c
        struct inode *ip;
        if(omode & O_CREATE){
            ip = create(path, T_FILE, 0, 0);
            if(ip == 0) {
                end_op();
                return -1;
            }
        }
    ```

* If the flags are passed for only read and write system call then the
  accrodingly the **inode is found** using the path name. Also a check is
  preformed on the inode if it is directory inode.

    ```c
        else {
            if((ip = namei(path)) == 0){
                end_op();
                return -1;
            }
            ilock(ip);

            /* if it is directory inode and opened in read mode then abort transaction */
            if(ip->type == T_DIR && omode != O_RDONLY){
                iunlockput(ip);
                end_op();
                return -1;
            }
        }
    ```

* Now find the file pointer in the **kernel's file table** where all the file 
  structure pointers are stored for the system.

    ```c
        struct file *f;
        f = filealloc();        /* kernel's file table pointers */
        fd = fdalloc(f);        /* file descripters in the open file table per process */
    ```
    
* Now change the attributes of the file which is found, accordingly to 
  to the given permission and inode
    ```c
        f->type = FD_INODE;
        f->ip = ip;
        f->off = 0;
        f->readable = !(omode & O_WRONLY);
        f->writable = (omode & O_WRONLY) || (omode & O_RDWR);
        return fd;
    ```







