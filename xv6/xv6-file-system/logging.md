# MIDDLE LAYER FILE SYSTEM HANDLING CODE

## LOGGING CODE

* Logging is mechanism of recovery from the physical disk.
    + recovery when there are inconsistent data structures of file system due 
      to power failure, umounting, dirty writes not written on disk.
    + multiple operations needed for system call (like open) for which some
      writes succeed and some don't

* In the log all the changes for a **transaction** (an operation) are either
  written completely or not at all.

* For recovery, completed / commited operations must be rerun, and incompleted 
  operations must be neglected.

* xv6 system call does not directly write the on-disk file system data
  structures, Rather a system call named begin\_op() and end\_op() are made 
  for starting and starting and commiting transaction.

    + begin\_op increaments log.outstanding.
    + end\_op decreaments log.outstanding and if it is 0, then calls, commit()

* During the code of system call if the cache buffer is modified, and done with 
    + log\_write() is called
    + This copies the block in an array of blocks inside log, the block is not
      written in it's actaul place in FS as of now.

* When finally commit is called, the modified block is written back to the disk.


## xv6 logging code

* Interestingly what xv6 does, write you want to do any modification to actaul
  hard disk, it creates transaction with begin\_op and ends transaction with end\_op. 
  
* The writes to disk are not done immediately, rather logs of such writes is 
  created, after which logs are affected in to do write in actual disk. This
  saves the file system from having inconsistent data structures.

* The structure of the log is as given below 

    ```c
        #define MAXOPBLOCKS  10  // max # of blocks any FS op writes
        #define LOGSIZE      (MAXOPBLOCKS*3)  // max data blocks in on-disk log

        struct logheader {
            int n;                          // index in the array block 
            int block[LOGSIZE];             // array of block numbers, currently being stored 
        };
        
        struct log {
            struct spinlock lock;           // lock 
            int start;                      // first log block on disk (starts with logheader)
            int size;                       // total number of log blocks : equal ot 30 
            int outstanding;                // how many FS sys calls are executing.
            int committing;                 // in commit(), please wait.
            int dev;                        // device number of hard disk
            struct logheader lh;            
        };
    ```

* The actual representation of the log file structure is as represented below.
  Note this representation is complete **fs.img**

```text
              log start <----+ struct +-> log start + 1            +--> log start + 30
                             |  log   |                            |
+------------+---------------+--------+----------------------------+---------+----------------------------------+
| boot block |  super block  |  log   |  52  | 68 | ...        |   |  log    |  inodes | bitmaps |  data blocks |   
+------------+---------------+--------+----------------------------+---------+----------------------------------+
                             |        |
    +------------------------+        +------------------------+
    |lock, start, size, etc  | n | 52  | 68 |    ...       |   | 
    +----------------------------------------------------------+
                               0   1    2        ...         29 (arrays of blocks)
```

* Now the when actual system call like sys\_write for writing block on disk,
  the changes actaully happen on the log area and not on the actaul file system,
  basically this helps in avoiding inconsistent data structures.
    
* Thus the **sys\_write** ultimately writes by using **log\_write** in the log
  area on the disk. The sequence of writes on log are done by creating and
  ending transaction.
    
    ```text
        begin_op()
        writei  ----> bread         (read data from disk)
                +---> log_write     (write modified data on disk)
        end_op()
    ```

* **Note : there can be multiple writes combined between read and write.**


### initlog function

* **initlog** is called **inside forkret**, when the init process is returning 
  to trapret. **Question - why called here why not called in main function ?**

    ```c
        void initlog(int dev);
    ```

* The first thing the initlog needs to do is to **read the super block**, since
  the superblock will give the initial starting address of the log block.

    ```c
        struct superblock sb;
        readsb(dev, &sb);
    ```

* After reading the super block initialize the parameters related to log structure.

    ```c
        log.start = sb.logstart;
        log.size = sb.nlog;
        log.dev = dev;
    ```

* Also the init process before executing will do **recovery from the log**, since
  init before executing, the **initlog** is called.

    ```c
        recover_from_log();
    ```

### begin\_op    
   
* The function is called before any transaction with the actual file system data.

    ```c
        // called at the start of each FS system call.
        void begin_op(void);
    ```

* There is infinite loop, which handles conditions like, if the **logs are
  currently been commited** i.e. written out on the file sytem disk. Also if
  **there are many system calls for modifying data on disk**, in such cases it 
  simply **makes the process sleep**.
    
    ```c
        while(1){

            // logs being commited on the disk currently 
            if(log.committing){
                sleep(&log, &log.lock);
            } 

            // to many system calls trying to begin transactions
            else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
                // this op might exhaust log space; wait for commit.
                sleep(&log, &log.lock);
            } 

            // The transaction starts by increamenting the outstanding value.
            else {
                log.outstanding += 1;
                release(&log.lock);
                break;
            }
        }
    ```


### end\_op

* The function call that ends the transaction with the file system.

    ```c
        // called at the end of each FS system call.
        // commits if this was the last outstanding operation.
        void end_op(void);
    ```

* The end transaction decreaments the outstanding value by 1, signifying 
  one transaction less.

    ```c
        log.outstanding -= 1;
    ```

* If the **value of outstanding in log struct is 0** then **commiting is set** 
  to one signifying that all the changes made in log will be updated on file 
  system. **Else some other process can be waked up** and given the access to 
  begin a file system transaction.

    ```c 
        int do_commit = 0;
        if(log.outstanding == 0){
            do_commit = 1;
            log.committing = 1;
        } else {
            // begin_op() may be waiting for log space,
            // and decrementing log.outstanding has decreased
            // the amount of reserved space.
            wakeup(&log);
        }
    ```

* Finally if commiting is set, then call to commit is made to actually make 
  the modified blocks update on the actaul file system.
    
    ```c
        if(do_commit){
            // call commit w/o holding locks, since not allowed
            // to sleep with locks.
            commit();
            log.committing = 0;
            wakeup(&log);
        }
    ```

### log\_write

* The function is called by many upper layer functions, when they are done with 
  modifying any paritcular buffer/blocks and want to actaully make changes to actual disk.

    ```c
        void log_write(struct buf *b);
    ```

* Iterates over the 30 entries of the logs, checking if buffer has already
  noted, is not noted, then the buffer block number is appended in the array.
  **Note : buffer data is not yet written, it is noted in log structure to 
  be written to the actual log space on disk.**  

    ```c
        for (int i = 0; i < log.lh.n; i++) {
            if (log.lh.block[i] == b->blockno)   // log absorbtion
                break;
        }

        log.lh.block[i] = b->blockno;
        if (i == log.lh.n)                      // append the block number in log blocks array 
            log.lh.n++;

        b->flags |= B_DIRTY;                    // marks as dirty
    ```


### commit 

* The function is only called when the last system call making transaction with
  the file system has ended, basically when **outstanding value in the log struct 
  becomes is zero** in the **end\_op**.

    ```c
        static void commit()            // static function in log.c
    ```
   
* Now commit calls **write\_log**, which writes the modified blocks from buffer
  cached to the log. The **write\_head** will write the log header on disk. 
  Next the **install\_trans** will reflect all the modified log blocks to
  actual file system blocks. Now finally, the **write\_head** with number of 
  log blocks set to zero initializes the log header to contain no blocks.

    ```c
        if (log.lh.n > 0) {
            write_log();     // Write modified blocks from buffer cache to log
            write_head();    // Write header to disk -- the real commit
            install_trans(); // Now install writes to home locations
            log.lh.n = 0;
            write_head();    // Erase the transaction from the log
        }
    ```


### write\_log

* Function is called only the **commit** function. This will actaully copy the 
  modified data blocks in buffer cache to the log area of the disk.

    ```c
        // Copy modified blocks from cache to log.
        static void write_log(void);
    ```

* The write happens from cache to log. The bread happens from the log.start to 
  log.end, so each block in the log is read, the corresponding entry from 
  the buffer cache is is copied to the pariticular log block in actaul disk.

    ```c
        for (int tail = 0; tail < log.lh.n; tail++) {
            struct buf *to = bread(log.dev, log.start+tail+1);      // log block
            struct buf *from = bread(log.dev, log.lh.block[tail]);  // cache block
            memmove(to->data, from->data, BSIZE);
            bwrite(to);  // write the log
            brelse(from);
            brelse(to);
        }
    ```

### write\_head

* The function writes the log header present at log.start just after the super
  block on to the disk. The log header contains the contains the array of blocks 
  which are modified.

    ```c
        static void write_head(void);
    ```

* The read the earlier log header which is stored at log.start. Remember the 
  read happens in the buffer cache of kernel.

    ```c
        struct buf *buf = bread(log.dev, log.start);
        struct logheader *hb = (struct logheader *) (buf->data);
    ```

* Now copy all the new modified blocks to the memory region which is read in
  buffer cache using the global log structure.

    ```c
        hb->n = log.lh.n;
        for (int i = 0; i < log.lh.n; i++) {
          hb->block[i] = log.lh.block[i];
        }
    ```
   
* Remember copy changes are refected by the in memory kernel cache and not 
  the actaul log header on the disk. Thus we write the buffer cache on disk.

    ```c
        bwrite(buf);
        brelse(buf);
    ```

### install\_trans

* This will copy the blocks from log to the actual appropriate file system blocks.
    
    ```c
        // Copy committed blocks from log to their home location
        static void install_trans(void);
    ```
   
* The buffer cache is used to read the data from the log block, and copy it
  to actual block which is part of file system read in another buffer cache.

    ```c
        for (int tail = 0; tail < log.lh.n; tail++) {
            struct buf *lbuf = bread(log.dev, log.start+tail+1);    // read log block
            struct buf *dbuf = bread(log.dev, log.lh.block[tail]);  // read file system block 
            memmove(dbuf->data, lbuf->data, BSIZE);                 // copy data between buffers
            bwrite(dbuf);                                           // write to the file system
            brelse(lbuf);
            brelse(dbuf);
        }   
    ```

