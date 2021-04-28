# UPPER LAYER FILE SYSTEM HANDLING CODE

## DIRECTORIES AND FILES 

* xv6 kernel maintains a global file structure which reflects all the files 
 persent on the file disk. It basically can array in xv6 of struct file.
    
    ```c
        // global file table array 
        struct {
            struct spinlock lock;           // spin lock to protect the array
            struct file file[NFILE];
        } ftable;
    ```

* The structure of the file or basically the file control block.

    ```c
        struct file {
            enum { FD_NONE, FD_PIPE, FD_INODE } type;   // FCB type
            int ref;                                    // reference count
            char readable;                              // can read
            char writable;                              // can write
            struct pipe *pipe;                          // is pipe file 
            struct inode *ip;                           // inode of file 
            uint off;                                   // offset of file
        };
    ```

* **Note** that the file type is different from inode type which can be DEVICE
  , FILE and ONDISK FILE. The FCB gives notion of open files by processes.


# FILE SYSTEM CALLS 

## open  

* The open system call should take filename and mode as input for openning a
  file and returns the file descripter which is an index in the ofile array.

* The actual work of open lies in parsing the file name, componentwise
  seperating the paths and during the parsing it needs to access the on disk
  inodes of directories and their data blocks.

* Finally it needs to create entry in file array structure of the process PCB 
  and returns the index of the opened file structure in the array.

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
            /* name is going to traverse the path and returns on disk inode */
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

## namei 

* The namei function takes input as string which is file path and traverses
  the file path untill the inode with paritcular file path is found and finally
  return the on disk inode of the file.

* The namei calls the namex function do the work of traversing the file path

    ```c
        struct inode* namei(char *path) {
            char name[DIRSIZ];
            return namex(path, 0, name);
        }
    ```

## namex

* The namex function is passed an file path along with integer nameiparent 
  which with set to non zero gets the iode of the parent of the file inode.

    ```c
        // Look up and return the inode for a path name.
        static struct inode* namex(char *path, int nameiparent, char *name);
    ```

* First it does a check if the path that is passed is absolute path name 
  or relative path name.

    ```c
        if(*path == '/')
            // simply gets in memory inode from inode cache and initiazes structure with given inode number
             ip = iget(ROOTDEV, ROOTINO);       
        else
            // idup simply increaments the reference count on the inode 
            ip = idup(myproc()->cwd);
    ```

* Now there is while which parses the given file name using parser named skipelem.
  The parser gets the first component of the name from the given path. 
  example if path = "/a/b/c" one call to skipelem get name as "a" and sets 
  path as "/b/c". Note skipelem is stops one level early in traversing i.e. 
  if path = "a" then name = "\0" and path = "a" and return 0;

    ```c
        while((path = skipelem(path, name)) != 0){
          
          // if inode is is not directory return 0;
          if(ip->type != T_DIR){
                return 0;
          }

          // nameiparent stops one level early 
          if(nameiparent && *path == '\0'){
            return ip;
          }

          // does directory lookup to get the next inode with 
          // name obtained by traversing the file name path 
          if((next = dirlookup(ip, name, 0)) == 0){
            return 0;
          }

          ip = next;
        }
        // finally returns the inode of given file path 
        return ip;
    ```

## dirlookup 

* The function looks up given name in the given directory inode and returns 
  inode for the name found in the directory.

    ```c
        // Look for a directory entry in a directory.
        // If found, set *poff to byte offset of entry.
        struct inode* dirlookup(struct inode *dp, char *name, uint *poff);
    ```

* The directory lookup calls the readi function return 

    ```c
        uint off, inum;
        struct dirent de;

        for(off = 0; off < dp->size; off += sizeof(de)){
            if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
                panic("dirlookup read");
            if(de.inum == 0)
                continue;
            if(namecmp(name, de.name) == 0){
                // entry matches path element
                if(poff){
                    *poff = off;
                    inum = de.inum;
                    return iget(dp->dev, inum);
                }
            }
        }
        return 0;
    ```





