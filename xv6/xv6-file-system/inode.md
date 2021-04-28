# MIDDLE LAYER FILE SYSTEM HANDLING CODE

## INODES 

* The structure which holds important attributes about the file/directory present 
  on the file system. Note that there is will difference between the actual 
  inode present on the disk, and in memory inode used by xv6 kernel.

* The on-disk inode structure is basically the attributed of the file present 
  on the physical disk.

    ```c
        // On-disk inode structure
        struct dinode {
            short type;           // File type
            short major;          // Major device number (T_DEV only)
            short minor;          // Minor device number (T_DEV only)
            short nlink;          // Number of links to inode in file system
            uint size;            // Size of file (bytes)
            uint addrs[NDIRECT+1];   // Data block addresses
        };
    ```

* The in-memory inode structure is used by the xv6 kernel for manipulating the
  file related I/O for proc structure used allocated to particular structure.
  Note the xv6 kernel maintains in memory global inode table for the 
  avaiable files on the file system.
    
    ```c
        // in-memory copy of an inode
        struct inode {
            uint dev;               // Device number
            uint inum;              // Inode number
            int ref;                // Number of process currently using inode
            struct sleeplock lock;  // protects everything below here
            int valid;              // inode has been read from disk?
        
            short type;             // copy of disk inode
            short major;
            short minor;
            short nlink;
            uint size;
            uint addrs[NDIRECT+1];
        };
    ```

* The kernel keeps a subset of on disk inodes, those in use, in memory as long 
  as the **ref** count is greate than zero.

### iget

* The function gets the inode for the particular inode number. **Note inode 
  doesn't read inode table from the disk**

    ```c
        // Find the inode with number inum on device dev
        // and return the in-memory copy. Does not lock
        // the inode and does not read it from disk.
        static struct inode* iget(uint dev, uint inum);
    ```


* The function simply searches the inode cache maintained by kernel to locate 
  the inode with the given particular number. If it founds the inode, then 
  it increaments the reference count return the pointer to inode.
    
    ```c
        struct inode *ip, *empty;
        // Is the inode already cached?
        empty = 0;
        for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++) {
            if(ip->ref > 0 && ip->dev == dev && ip->inum == inum) {
                ip->ref++;
                release(&icache.lock);
                return ip;
            }
            if(empty == 0 && ip->ref == 0)    // Remember empty slot.
                empty = ip;
        }
    ```

* If the function doesn't finds any entry in the inode cache, then creates
  rather appends the new entry in the empty inode slot.

    ```c
        ip = empty;             // empty inode slot
        ip->dev = dev;          // device number 
        ip->inum = inum;        // inode number 
        ip->ref = 1;            // first process to get the inode slot 
        ip->valid = 0;          // valid is set to zero, since inode is not read from disk 
        return ip;
    ```

* **Note : The inode doesn't need to have valid bit set and also the process can 
  share the same inode structure from the inode cache.**


### iupdate 

* Reads the disk block corresponding to the particular inode, updates/modifies 
  the specified inode on disk with the in memory buffer cache inode values. The
  function calls **log\_write** for updating the inode on disk.
    
    ```c
        // copies a modified in memory cached inode to the inode on disk
        void iupdate(struct inode *ip);
    ```

* Simply the reads the inode on disk into buffer cache, copy the values of in 
  memory cached inode into the buffer cache corresponding to the inode on disk,
  then simply writes the buffer cache using **log\_write**
    
    ```c
        struct buf *bp;
        struct dinode *dip;

        bp = bread(ip->dev, IBLOCK(ip->inum, sb));
        dip = (struct dinode*)bp->data + ip->inum%IPB;
    ```

* Update the values of in memory kernel cached inode to the buffer cache
  corresponding to the actual inode on disk.

    ```c
        dip->type = ip->type;
        dip->major = ip->major;
        dip->minor = ip->minor;
        dip->nlink = ip->nlink;
        dip->size = ip->size;
        memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
    ```

* Write the updated buffer cache on to the actual disk using log write and free
  the buffer cache used to update the inode struct on disk.
    
    ```c
        log_write(bp);
        brelse(bp);
    ```

### ilock

* This acquires a locks on the in-memory inode and reads the actual on disk inode 
  value to the in-memory cached inode. Thus any process which needs to read
  inode will needs to do **iget** followed by **ilock**.

    ```c
        void ilock(struct inode *ip);
    ```

* Checks if the valid bit of the inode is not set, and thus creates the
  corresponding inode on disk, to the in-memory cache inode.

    ```c
        struct buf *bp;
        struct dinode *dip;

        if(ip->valid == 0) {
            bp = bread(ip->dev, IBLOCK(ip->inum, sb));
            dip = (struct dinode*)bp->data + ip->inum%IPB;
            ip->type = dip->type;
            ip->major = dip->major;
            ip->minor = dip->minor;
            ip->nlink = dip->nlink;
            ip->size = dip->size;
            memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
            brelse(bp);
            ip->valid = 1;
        }
    ```

* The function exact reverse of **iupdate**.


### iput

* The function decrements the reference count of the inode, and if the
  reference count becomes the valid bit is also set to zero. Basically its
  code for giving away paritcular inode.

    ```c
        void iput(struct inode *ip);
    ```

* Checks if the valid bit is set and it's only one process holding the inode, 
  thus updates the ondisk inode accordingly and sets the valid bit to zero again.

    ```c
        if(ip->valid && ip->nlink == 0) {
            int r = ip->ref;
            if(r == 1) {
                // inode has no links and no other references: truncate and free.
                itrunc(ip);
                ip->type = 0;
                iupdate(ip);
                ip->valid = 0;
            }
        }
    ```

* Decrements the reference count corresponding to the inode.

    ```c
        ip->ref--;
    ```

### itrunc

* The function writes the data block of the inode on to the disk.

    ```c
        static void itrunc(struct inode *ip);
    ```
  

    ```c
        int i, j;
        struct buf *bp;
        uint *a;

        for(i = 0; i < NDIRECT; i++){
            if(ip->addrs[i]){
                bfree(ip->dev, ip->addrs[i]);
                ip->addrs[i] = 0;
            }
        }
    ```


    ```c
        if(ip->addrs[NDIRECT]){
            bp = bread(ip->dev, ip->addrs[NDIRECT]);
            a = (uint*)bp->data;
            for(j = 0; j < NINDIRECT; j++){
                if(a[j])
                    bfree(ip->dev, a[j]);
            }
            brelse(bp);
            bfree(ip->dev, ip->addrs[NDIRECT]);
            ip->addrs[NDIRECT] = 0;
        }

        ip->size = 0;
        iupdate(ip);
    ```


