## KINIT1 FUNCTION (contructor for memory management)

* **kinit1 function** : basically its the **initialization of the OS Memory
  management data structures** which manages the operation over list of free 
  frames on the RAM.

    + kinit1 is called with arguments as
        ```c
            extern char end[]; // first address after kernel loaded from ELF file
            
            kinit1(end, P2V(4*1024*1024)); // phys page allocator
        ```

        * The **end** is global variable initialized by the linker which **points 
          to last address** of the kernel code loaded from ELF file. The virtual
          address pointed by end -> 801154a8

        * **P2V is marco** which converts the phyical address to virtual address,
          basically it adds KERNBASE address value in the physicall address.

            ```c
                #define KERNBASE 0x80000000         // First kernel virtual address
                #define P2V(a) ((void *)(((char *) (a)) + KERNBASE))
            ```
            + **Question : why convert physical to virtual ?**

                + Remember of setting up of Page Directory Table (PDE), before
                  entering the main function of kernel, so now every address
                  issued by CPU passes through the MMU PDE table. Thus now if
                  kernel want's to access any of the physical page address
                  it needs reverse the mapping of virtual to physical addresses.
            
                + **kernel.ld** file had linked all the kernel code at location 
                  0x80100000, which is nothing but KERNBASE + EXTMEM. Thus
                  every address inside the code is virtual address.

                + **Work of P2V macro is just to add KERNBASE so that virtual 
                  address issued is to the corresponding 512 entry in PDE Table
                  which is maps the virtaul memory address to physical frame 0**
        
        * The memory range that the kinit1 function is passed as argument is 
            
            |    Argument    | Hexadecimal virtual address | Hexadecimal physical address | decimal value |
            |----------------|-----------------------------|------------------------------|---------------|
            |    **end**     |         0x801154a8          |           0x001154a8         |   1135784     |
            | **P2V(4 MB)**  |         0x80400000          |           0x001154a8         |   4194304     |
        
            Memory space of about 2.9168319702 MB ~ 3 MB is used as free frame 
            region for initialization of the kernel's memory management data structures.

    + **kinit1** function definition is given below, which further calls **freerange**

        ```c
            void kinit1(void *vstart, void *vend) {
                freerange(vstart, vend);
            }
        ```

    + **freerange** does the work of actaully freeing all the frames in between
      between the given range of addresses (vstart and vend).
        
        ```c
            #define PGSIZE          4096    // bytes mapped by a page
                
            void freerange(void *vstart, void *vend) {
                char *p;
                p = (char*)PGROUNDUP((uint)vstart);
                for(; p + PGSIZE <= (char*)vend; p += PGSIZE)
                    kfree(p);
                }
        ```

        + **Question : what exactly is meant by freeing the frames ? who does it ?**
            + Making frames free is manipulating the OS data structures which
              maintains the free list of frames in memory.
            + kernel memory management function do this job.
              
                 |   function   |           work            | 
                 |--------------|---------------------------|
                 | **kfree**    | free's any given frame    |
                 | **kalloc**   | allocates any free frame  |

    + **Question : The page table is set up for the 4 MB pages but the free
      frame initialization is done for 4KB pages ?**

        + The 4 MB pages size entries in PDE were temporary, now we would
          actually going to change the single level paging to two level paging.

        + Remeber the **PTE_PS** bit which as set up by entrypgdir before
          jumping to the main function will now be turned off.
            + PTE_PS bit = 0 ---> 4 KB pages
            + PTE_PS bit = 1 ---> 4 MB pages

