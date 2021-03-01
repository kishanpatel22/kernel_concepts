# xv6 KERNEL MEMORY MANAGEMENT 

* For memory management, xv6 sets up segmentation effectively (before bootmain), 
  then sets up paging (before kernel init). 
  + Note that **segmentation is effectively bypassed** in xv6 by mapping all the
    memory range using the segment selector and limit register.
  + Note that **page table consits of only 2 Page directory entry** both mapping 
    to the same 0th frame in physical memory.

* The function **kinit1**, **kinit2** are contructors used for memory management
  data structures in xv6. Basically xv6 provides two functions **kfree** and
  **kalloc** to effectively manage the frames in physical memory.


## xv6 free frame list management 

* The **kernel needs to maintains list of free frames in memory** to allocate
  frames to various process, on demand I/O, etc, thus it has to use some data
  structure that allows insert and delete operations on free frames in memory 

* xv6 uses simple **singly link list** data structure for maintaining **stack of
  free frames**. The memory mangement data structures for free frame management
  are defined in **kalloc.c** file. 

* The memory management data structures are defined as given below
    ```c
        /* the singly link list data structure consisting
         * of self referential pointer to next free frame.
         */
        struct run {
            struct run *next;
        };
        
        /* global structure variable kmem consisting of pointer  
         * called as freeframe to the head of the link list.
         */
        struct {
            struct spinlock lock;
            int use_lock;
            struct run *freelist;
        } kmem;
    ```

* Now simiply two functions that manipulate this data structure are written in 
  the file **kalloc.c**

    + Adding more free frames in the list -> **kfree** 
        ```c
            void kfree(char *v) {
                struct run *r;

                // Fill with junk to catch dangling refs.
                memset(v, 1, PGSIZE);

                r = (struct run*)v;
                r->next = kmem.freelist;

                kmem.freelist = r;
            }
        ```
        * kfree function adds the free frame list at the head of the existing
          free frame list, basically geting a stack like push behaviour.


    + Allocating free frames from the list -> **kalloc**
        ```c
            char* kalloc(void) {
                struct run *r;

                r = kmem.freelist;
                if(r)
                    kmem.freelist = r->next;

                return (char*)r;
            }
        ```
        * kalloc function returns the first element from the list of free
          frames, basically getting a stack like pop behaviour.





