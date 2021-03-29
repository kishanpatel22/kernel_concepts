## KVMALLOC FUNCTION (KERNEL PAGE TABLE SET UP)

* Note before entrying the main function of kernel, the page table set up was
  done by **entry routine**, basically setting up **entrypgdir** and settting
  the CR3 register accordingly. This was basically Page Directory Tabel set up 
  (one level paging) for 4 MB pages. 

* Now in **kvmalloc function** which is defined in **vm.c**, The page table
  entry would be changed to constitute **4 KB pages**, and setting up **two level
  paging** with **PDE** and **PTE** to reference physical frame in memory.
  Basically the function sets up the page table for the kernel.

    ```c
        typedef uint pde_t;

        /* global pointer to the kernel page directory */
        pde_t *kpgdir;  
        
        void kvmalloc(void) {
            kpgdir = setupkvm();
            switchkvm();
        }
    ```

### SETUPKVM function : set up the two level paging for kernel

* **setupkvm** function returns the address of kernel page directory location,
  which is set up using the two level paging scheme i.e. PDE points to PTE
  which further points to actual frame in memory. Setupkvm returns the starting 
  address of the PDE table.

    ```c
        // Set up kernel part of a page table.
        pde_t* setupkvm(void) {
            
            pde_t *pgdir;

            /* structure which defines two level paging scheme for kernel   */
            struct kmap *k;
            
            /* allocate 4 KB for setting the page directory table           */
            if((pgdir = (pde_t*)kalloc()) == 0) {
                return 0;
            }
            
            /* initialize the 4 KB chunk of memory with all values as zero  */
            memset(pgdir, 0, PGSIZE);
            
            /* loop through to setup 4 page directory entires               */
            for(k = kmap; k < &kmap[NELEM(kmap)]; k++) {
                if(mappages(pgdir, k->virt, k->phys_end - k->phys_start,
                            (uint)k->phys_start, k->perm) < 0) {
                    freevm(pgdir);
                    return 0;
                }
            }

            return pgdir;
        }
    ```
        
    + The setupkvm function attempts to allocate free frame using kernel's kmalloc 
      function, which returns pointer to free frame in memory of 4 KB which is
      used as Page Directory Table initialized with 4 Page Directory Entries.
    
    + The memset function is used to initialize the portion of the memory
      allocated to fixed values like zero.

    + Now next there is loop which initializes the kernel page table entries
      from global array variable called **kmap**, by looping through all
      elements in the array **kmap**.

        ```c
            static struct kmap {
                void *virt;
                uint phys_start;
                uint phys_end;
                int perm;
                } kmap[] = {
                    { (void*)KERNBASE, 0,             EXTMEM,    PTE_W}, // I/O space
                    { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0},     // kern text+rodata
                    { (void*)data,     V2P(data),     PHYSTOP,   PTE_W}, // kern data+memory
                    { (void*)DEVSPACE, DEVSPACE,      0,         PTE_W}, // more devices
                };  
            }
        ```

        * Note : kmap is global variable which is array of type structure
          named as kmap, defined with data members as
            + virtual address
            + physical start address
            + phyiscal end address
            + permisions 
        
        * Note : **V2P is macro** for translating the virtual to physical addresses 

            ```c
                #define V2P(a) (((uint) (a)) - KERNBASE)
            ```

        * Address values for data members at each index in array of struct kmap

            ```c
                #define EXTMEM  0x100000            // Start of extended memory
                #define PHYSTOP 0xE000000           // Top physical memory
                #define DEVSPACE 0xFE000000         // Other devices are at high addresses

                #define KERNBASE 0x80000000         // First kernel virtual address
                #define KERNLINK (KERNBASE+EXTMEM)  // Address where kernel is linked
            ```

        * The data is global variable which is pointer defined by the linker
          and can be found in the **kernel.sym** file.

          ```text
                80108000 data
          ```

    + Basically what are we trying to achieve by such type of memory mappings for kernel 
        ![kmap memory mapping](./images/kernal_paging_maps.jpg) 

        |     region in memory      |       start    |      end      |  size        |   permissions   |
        |---------------------------|----------------|---------------|--------------|-----------------|
        |         I/O space         |    0           | EXTMEM        |   1 MB       |     write       |
        |   kernel code + RO data   |  EXTMEM        | EXTMEM + data |   0.03125 MB |     read        |
        |   kernel data + memory    |  EXTMEM + data | PHYSTOP       |   223 MB     |     write       |
        |        devspace           |  DEVSPACE      | END           |   4 MB       |     write       |

    + **mappages** function does the work of allocating the Page Table entry 
      for any given the Page Directory Entry with address, size and permission.

        * Note the parameters passed to the function are 
            + **address of the start of page directory**
            + **starting virtual address**
            + **size of the region**
            + **starting physical address**
            + **permission**

        ```c
            static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm) {
                char *a, *last;
                pte_t *pte;
                
                /* starting virtual address                                 */
                a = (char*)PGROUNDDOWN((uint)va);

                /* ending virtual addresss                                  */
                last = (char*)PGROUNDDOWN(((uint)va) + size - 1);
                
                /* loop through to allocate the PTE entries in PDT          */
                for(;;) {

                    /* walkpgdir basically scans the page directory table and
                       check if the given entry was already present or not if
                       not then it allocate the 4 KB free memory and marks it
                       as PTE for appropriate location in PDT.
                     */
                    if((pte = walkpgdir(pgdir, a, 1)) == 0)
                        return -1;

                    if(*pte & PTE_P)
                        panic("remap");

                    /* explicitly set up the permission for the PTE entry   */ 
                    *pte = pa | perm | PTE_P;

                    if(a == last)
                        break;
                    a += PGSIZE;
                    pa += PGSIZE;
                }
                return 0;
            }
        ```

        * The function simply calls **walkpgdir** which checks if there exits
          already page table entry for the particular virtual address, and if
          not then allocates it. After the successful allocation the the
          physical address, permision and present bits are set in the entry.
        
        ```c
            static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc) {
                pde_t *pde;
                pte_t *pgtab;
                
                pde = &pgdir[PDX(va)];
                if(*pde & PTE_P){
                    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
                } else {
                    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
                        return 0;
                    // Make sure all those PTE_P bits are zero.
                    memset(pgtab, 0, PGSIZE);
                    // The permissions here are overly generous, but they can
                    // be further restricted by the permissions in the page table
                    // entries, if necessary.
                    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
                }
                return &pgtab[PTX(va)];
            }
        ```
        
        * **PDX macro** basically gets the upper 10 bits from the given address,
          the upper 10 bits signifiy the index in the PDE table.

        ```c
            #define PDX(va)         (((uint)(va) >> PDXSHIFT) & 0x3FF)
        ```
        
        * **Note** : the permission set by the wakepgdir are read and write, although
          the function calling walkpgdir can modify the entry accordingly. Also
          if the alloc is set 1 implies, that the if entry not found in dir it
          will be made in the page directory table.
    
* **switchkvm** basically changes the CR3 register which now points to the new 
  Page Descripter Table which is setup for the 4 KB pages using two level paging.

```c
    // Switch h/w page table register to the kernel-only page table,
    // for when no process is running.
    void switchkvm(void) {

        /* change the CR3 register to the point to PDT setup for kernel     */
        lcr3(V2P(kpgdir));   // switch to the kernel page table

    }
```

