# LOWER LAYER FILE SYSTEM HANDLING CODE

## DISK DRIVER (IDE - Intergrated Drive Electronics) 

* The driver code provides an API which helps the software to actaully manipulate
  the hardware resources. The **disk driver** helps in actual lower level 
  read and write data on the physical disk.

* In xv6, there is file named **ide.c** which contains the disk driver code.


## Physical hard disk obtained by Makefile 

* There are two harddisk which are complied in the recipe written in Makefile.
  Note only the disk 0 is present on the disk 1, thus all the read/write must
  happen to the disk 0. Note **disk0 is xv6.img** and **disk1 is fs.img**.

    ```sh
        # the hard disk containing the xv6 kernel image 
        xv6.img: bootblock kernel
	        dd if=/dev/zero of=xv6.img count=10000
	        dd if=bootblock of=xv6.img conv=notrunc
	        dd if=kernel of=xv6.img seek=1 conv=notrunc
        
        # the hard disk containing the file system of xv6
        fs.img: mkfs README $(UPROGS)
	        ./mkfs fs.img README $(UPROGS)
    ```

* Not that any read / write from the disk is stored in the xv6 kernel's buffer 
  cache which basically caches the one block contents which are read / write.
  The buffer bridges the gap between the actaul harddisk read / write and
  process calling for read / write.

    ```c
        struct buf {
            int flags;                  /* flags for blocks : valid or dirty */
            uint dev;                   /* which ide to read write from */
            uint blockno;               /* block number which is read / write */
            struct sleeplock lock;
            uint refcnt;                /* number of process currently usng buffer */
            struct buf *prev; // LRU cache list
            struct buf *next;
            struct buf *qnext; // disk queue
            uchar data[BSIZE];          /* block size data for read / write */
        };
    ```

* The **dev**, device number and **blockno**, block number together determine 
  the particular sector on the physical hard disk.

* The xv6 kernel **buffers caches** are maintained in **singly link list** data 
  structure as wait queue implementation.

### ideinit 

* The function is used at the begining to initialize the physical hard disk.
  Note that the function is called only once by main fucntion of the kernel 
  before even creating the first init process.

    ```c 
        void main() {
            /* .... */
            ideinit();       // disk controller initialization
            /* .... */
        }
    ```

* The ideinit function is defined in **ide.c** as given below 

    ```c 
        void ideinit(void)
    ```

* ideinit function waits for the disk controller to responds and after which it 
  checks if the disk 1 is present or not by issuing a few instruction on disk
  port of the controller. Note that the disk number 1 is setup in the Makefile 
  for qemu virtual hardware initialization.

    ```c
        idewait(0);
        // Check if disk 1 is present
        outb(0x1f6, 0xe0 | (1<<4));
        for(i=0; i<1000; i++) {
            if(inb(0x1f7) != 0) {
                havedisk1 = 1;      /* disk 1 : fs.img is present as hardware */
                break;
            }
        }
    ```

* Lastly disk controller is switched back to the disk 0 which the **xv6.img** 
  which contains xv6 kernel.

    ```c
        // Switch back to disk 0.
        outb(0x1f6, 0xe0 | (0<<4));
    ```


### iderw

* The function is called everytime when we need to input / output from the disk.
  The buffer passed as parameter to the disk read / write driver code takes
  input as kernel's buffer cache. The buffer cache 

    ```c
        void iderw(struct buf *b)
    ```

* The buffer passed to the lower layer driver code will do check on buffer for 
  it being valid and dev number must set appropriately.
    
    ```c
        /* needs to lock before reading from the disk */
        if(!holdingsleep(&b->lock))
            panic("iderw: buf not locked");
        
        /* if block is valid and dirty bit is not set, 
         * then no need to proceed with reading or writing 
         */
        if((b->flags & (B_VALID|B_DIRTY)) == B_VALID)
            panic("iderw: nothing to do");
        
        /* check the device number for the buffer */
        if(b->dev != 0 && !havedisk1)
            panic("iderw: ide disk 1 not present");
    ```

* The **idequeue** is queue of the struct of type xv6 kernel buffers which are
  waiting for disk I/O to be completed. Now the iderw appends the current
  xv6 kernel buffer to the queue 

    ```c
        struct buf **pp;
        b->qnext = 0;                   /* next pointer marked as NULL */
        
        /* append b to the queue for buffer waiting for I/O */
        for(pp=&idequeue; *pp; pp=&(*pp)->qnext) { //DOC:insert-queue
            ;
        }
        *pp = b;
    ```

* However if the buffer appended was the **first buffer on queue list**, then 
  it will **start the disk driver controller** for read / write.

    ```c
        /* Start disk if necessary */
        if(idequeue == b) {
            idestart(b);
        }
    ```

* Now for the current kernel buffer wait untill the actual I/O is over, thus
  make the current process sleep by suspending or waiting.

    ```c
        /* Wait for request to finish */
        while((b->flags & (B_VALID|B_DIRTY)) != B_VALID) {
            sleep(b, &idelock);
        }
    ```


### idestart

* The function is called when **disk driver needs to start disk controller**
  for read / write and return. Note that there is not seperate read / write 
  flag send to idestart, the read / write is decided by dirty bit set in buffer.

    ```c
        // Start the request for b.  Caller must hold idelock.
        static void idestart(struct buf *b);
    ```

* Now the idestart does the work of **identifying the actual sector number** on 
  hard disk, the **commands** that need to be send to the actaul hard disk to 
  read or write accordingly.

    ```c
        /* sector per block on the disk */
        int sector_per_block =  BSIZE/SECTOR_SIZE;

        /* actual sector on the disk */
        int sector = b->blockno * sector_per_block;

        /* read command to controller */
        int read_cmd = (sector_per_block == 1) ? IDE_CMD_READ :  IDE_CMD_RDMUL;

        /* write command to controller */
        int write_cmd = (sector_per_block == 1) ? IDE_CMD_WRITE : IDE_CMD_WRMUL;
    ```

* Now the next subsequent instructions are **in and out instructions to the
  I/O ports of actual physical harddisk**. The instructions help in indicating
  the hard disk controller about the command that will send later.
    
    ```c
        idewait(0);
        outb(0x3f6, 0);  // generate interrupt
        outb(0x1f2, sector_per_block);  // number of sectors
        outb(0x1f3, sector & 0xff);
        outb(0x1f4, (sector >> 8) & 0xff);
        outb(0x1f5, (sector >> 16) & 0xff);
        outb(0x1f6, 0xe0 | ((b->dev&1)<<4) | ((sector>>24)&0x0f));
    ```

* Depending the upon the **dirty bit** set in the flags of the buffer, the 
  **read and write** is decided accoridingly. Note is case of the read with 
  dirty bit set will cause the kernel to pannic in **idearw** itelf.
    
    ```c
        /* write if dirty bit is set */
        if(b->flags & B_DIRTY){
            outb(0x1f7, write_cmd);
            outsl(0x1f0, b->data, BSIZE/4);
        } 
        /* read if dirty bit is set */
        else {
            outb(0x1f7, read_cmd);
        }
    ```

* **Note** : the idestart doesn't wait for the I/O to get completed, the control 
  is transferred to the **iderw** which has called **idestart**, to make the
  process sleep and wait I/O queue of kernel buffers.


### ideintr

* The interrupt handler which gets called when there is read / write from disk 
  is completed by the **disk controller**. The simple job of interrupt handler 
  is wakeup the process which was sleeping and schedule the next disk I/O if any.

* The **disk controller** after completing the disk I/O, will raise hardware 
  interrupt to the microprocessor, control will be transferred to the 
  **vector interrupt routine**, which jumps to **alltraps**, and futher calling
  the **trap function** to handle the disk controller interrupt.
    
    ```c
        /* trap.c file : handling the disk controller interrupt */
        case T_IRQ0 + IRQ_IDE:
            ideintr();              /* disk controller interrut handler */
            lapiceoi();
            break;
    ```

* The ide interrupt handler is defined in the **ide.c** file as given below

    ```c
        // Interrupt handler.
        void ideintr(void);
    ```

* xv6 assumes that the **interrupt that occured by the disk controller was 
  for the first buffer in the wait idequeue**. It is very simple mechanism
  adopted by xv6 for resolving the disk controller interrupts, thus the first
  process to make disk I/O will be the first process to accomplish the disk I/O.

    ```c
        struct buf *b;
        b = idequeue;       /* queue for all waiting buffers for disk I/O */
        
        /* First queued buffer is the active request */
        idequeue = b->qnext;
    ```

* Now if the buffer was for read command to controller, which is identified
  using the dirty bit in the flag of the buffer, the actaul data is read from
  the port of the disk using assembly language instruction.

    ```c
        // Read data if needed.
        if(!(b->flags & B_DIRTY) && idewait(1) >= 0) {
            insl(0x1f0, b->data, BSIZE/4);
        }
    ```

* Changing the **flags for the kernels buffer cache** after the I/O is completed.

    ```c
        b->flags |= B_VALID;
        b->flags &= ~B_DIRTY;
    ```

* Now remember the process after calling **iderw** was suspended or rather made
  to sleep, thus now the same process needs to be wakenup and made RUNNABLE 
  for execution.

    ```c
        // Wake process waiting for this buf.
        wakeup(b);
    ```

* Now subsequently, xv6 schedules disk I/O for the next buffer in the wait queue.

    ```c
        // Start disk on next buf in queue.
        if(idequeue != 0)
            idestart(idequeue);
    ```


### idewait

* The function contains a loop which simply waits for read / write I/O to get over.

