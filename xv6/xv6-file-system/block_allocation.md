# LOWER LAYER FILE SYSTEM HANDLING CODE

## BLOCK ALLOCATION


* The block allocation code is present in the **fs.c** file

### balloc

* Looks for block whose bitmap is zero indicating its free, and returns 
  the corresponding block number, by set the bit to one and calls **log\_write**.

    ```c
        // Allocate a zeroed disk block.
        static uint balloc(uint dev);
    ```

* **Iterating over all the blocks present in the bitmap blocks** of file system.
  For **each blocks iterating each byte to find at particular bit which is not 
  set** then its marks it as set, and returns the block number of corresponding
  to the new allocate block on disk. **Note - the updated bitmap block is
  written by lower layer function log\_write**
    
    ```c
        int b, bi, m;
        struct buf *bp;
        bp = 0;
        for(b = 0; b < sb.size; b += BPB) {
            bp = bread(dev, BBLOCK(b, sb));
            for(bi = 0; bi < BPB && b + bi < sb.size; bi++) {
                m = 1 << (bi % 8);
                if((bp->data[bi/8] & m) == 0) {     // Is block free?
                    bp->data[bi/8] |= m;            // Mark block in use.
                    log_write(bp);
                    brelse(bp);
                    bzero(dev, b + bi);
                    return b + bi;
                }
            }
            brelse(bp);
        }
    ```


### bfree

* Finds the right bitmap block clear the set bit, call **log\_write** to write
  the free block updated bitmap on disk.    

    ```c
        // Free a disk block.
        static void bfree(int dev, uint b);
    ```


* Finds the particular block which is present in the data block bitmaps which
  corresponds to the particular data block b, Updates the particular bit 
  entry in the block, making it zero, indicating a free block. **Note - the 
  updated bitmap block is written by lower layer function log\_write**
    
    ```c
        // finds the block in bitmap blocks which corresponds to particular 
        // block number in the data block region of the file system
        #define BBLOCK(b, sb) (b/BPB + sb.bmapstart)      
        
        struct buf *bp;
        int bi, m;
        
        // read the corresponding block from bitmap blocks
        bp = bread(dev, BBLOCK(b, sb));
        bi = b % BPB;
        m = 1 << (bi % 8);

        // update the pariticular bit corresponding to data block 
        bp->data[bi/8] &= ~m;

        // write the udapted bitmap blocks on disk
        log_write(bp);
        brelse(bp);
    ```

