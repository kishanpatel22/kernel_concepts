# LOWER LAYER FILE SYSTEM HANDLING CODE

## Buffer Cache 

* The buffer cache is implemented in xv6 for **caching the data flow with respect
  to physical disk**, thus there by providing the upper layers functionality 
  to do disk I/O from the data in cached buffer instead for any raw disk I/O.

* The advantage of having buffer cache is **speed up the execution time** for
  processes which are waiting for disk I/O by **reducing the unecessary reads 
  from the disk controller**.

* The buffer caches in xv6 are maintained using **doubly circular link lists**
  data structures manipulated over array for implementating MRU.

* The buffer cache code is present inf **bio.c** file.

* xv6 kernel maintains a global cache structure as given below

    ```c
        struct {
            struct spinlock lock;
            struct buf buf[NBUF];           /* array of buffer caches */

            // Linked list of all buffers, through prev/next.
            // head.next is most recently used.
            struct buf head;                /* head of doubly circular link list */
        } bcache;
    ```

### binit

* Initializes the doubly circular link list data structure. The head contains 
  the start of the link list with the next and previous pointers.

    ```c
        void binit(void) {
            struct buf *b;
    
            // Create linked list of buffers
            bcache.head.prev = &bcache.head;
            bcache.head.next = &bcache.head;
            for(b = bcache.buf; b < bcache.buf+NBUF; b++){
                b->next = bcache.head.next;
                b->prev = &bcache.head;
                bcache.head.next->prev = b;
                bcache.head.next = b;
            }
        }
    ```

### bget

* The function will **get struct buffer cache for given paritcular block number
  and device number**. Note the following function searches for the buffer
  cache in the list of buffer caches stored by the kernel, if not found then a
  new buffer cache is allocated.

    ```c
        static struct buf* bget(uint dev, uint blockno)
    ```

* Now searching for the buffer cache with given block number and device number
  happens by traversing the **doubly circular link list**

    ```c 
        struct buf *b;
        // Is the block already cached?
        for(b = bcache.head.next; b != &bcache.head; b = b->next){
            /* searching for the paritcular buffer cache */
            if(b->dev == dev && b->blockno == blockno){
                b->refcnt++;
                /* buffer cache is found */
                return b;
            }
        }
    ```
   
* If no buffer cache exists with the given block number and device number, then
  allocate a new buffer for the cache by finding a buffer cache which has link 
  count as zero and dirty bit not set, basically an unused buffer.

    ```c
        for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
            if(b->refcnt == 0 && (b->flags & B_DIRTY) == 0) {
                b->dev = dev;
                b->blockno = blockno;
                b->flags = 0;
                b->refcnt = 1;
                release(&bcache.lock);
                acquiresleep(&b->lock);
                return b;
            }
        }
    ```

### bread

* The function will **read the data from the buffer** with the given device 
  number and block number.

    ```c
        struct buf* bread(uint dev, uint blockno)
    ```

* Before even reading the buffer cache must be allocated for reading using the
  **bget** function

    ```c
        struct buf *b;
        b = bget(dev, blockno);
    ```

* **Piece of Magic** : Now check for valid bit in the flags, if the valid bit
  is set, then there is not need to read from the disk, one can directly refer 
  the data present in the block, however if valid bit is not set then we need
  to schedule disk I/O using iderw function.

    ```c
        /* this condition speeds up the execution time */
        if((b->flags & B_VALID) == 0) {
          iderw(b);                         /* read using the device driver */
        }
        return b;
    ```

### bwrite 

* The function will write the data from the buffer to disk. It uses the lower 
  layer **iderw** function writing by setting up the dirty flag in buffer cache.
    
    ```c
        void bwrite(struct buf *b) {
            b->flags |= B_DIRTY;            /* set the dirty bit before writing */
            iderw(b);                       /* write using the devide driver */
        }
    ```

### belse 

* **Releases the buffer cache** for the given processes. However note that there 
   is no deallocating buffer cache, instead the data and the device number and
   block number are kept intact, assuming there would be need in future.

    ```c
        // Release a locked buffer.
        // Move to the head of the MRU list.
        void brelse(struct buf *b) {
            b->refcnt--;
            if (b->refcnt == 0) {                   
                // no one is waiting for it deallocate it.
                b->next->prev = b->prev;
                b->prev->next = b->next;
                b->next = bcache.head.next;
                b->prev = &bcache.head;
                bcache.head.next->prev = b;
                bcache.head.next = b;
            }
        }
    ```

* Note the data is still there however the node is moved at the head of the 
  doubly link list.


