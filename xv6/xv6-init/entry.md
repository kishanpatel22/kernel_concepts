# xv6 KERNEL ENTRY function (entrying to kernel code)

* The bootmain function after loading the xv6 kernel at location 0x10000 in 
  RAM passes the control to the entry/start function which specified in the ELF 
  file of the kernel image. The last line of the bootmain function 

    ```c
        // Call the entry point from the ELF header.
        // Does not return!
        entry = (void(*)(void))(elf->entry);
        entry();
    ```

* The entry/start function of the kernel is present in the **entry.S** file.
  Note - kernel doesn't start execution from main, but rather starts execution
  from entry assembly language code.

* **Question : where exactly the control is transfered in memory ?**

    + The **kernel.ld** file contains the linker script, which during
      compilation will be interpreted by linker, and links the kernel at the
      memory location 0x80100000 (virtual address after setting up pagging). 
      
    + However the actual physical address of entry function is 0x0010000c
      because thats where the bootmain had copied the kernel contents in
      actual main memory without any pagging and effectively bypassing
      segmentation.

    + **Note** : there kernel is linked at location 0x80100000, the function 
      the elf->entry() is located at 0x8010000c virtual address. 
      (check **kernel.sym** file which is formed after compilation)


## ENTRY FUNCTION (Begining of kernel)

* **Setting up PAGING** for memory management.

    + The in x86 architecture the **PDE / PDT entries** refer to the structure 
      of the entries in two level paging. 
        + First level page entry - Page directory entry (PDE)
        + Second level page entry - Page Table entry (PTE)

    + ![PDE/PDT](https://www.viralpatel.net/taj/tutorial/image/page_table_entry.gif)

        + For the 4 KB pages 

            ```text
                                            Linear Address (32 bits)

                          10 bits                   10 bits                 12 bits
                 <-----------------------> <-----------------------> <--------------------> 
                31                      22 21                     12 11                   0
                |___page_directory_index__|_______page_table________|_______offset________|
                            |                         |                        |
                            |                         |                        |
            CR3 ----> +-----------+       +---> +----------+        +----> +----------+
               points |   Page    |       |     |   Page   |        |      |   4 KB   |
                      | Directory |-------+     |   Table  |--------+      |   Page   |
                      +-----------+             +----------+               +----------+

            ```

        + For the 4 MB pages

            ```text
                                            Linear Address (32 bits)

                          10 bits                               22 bits          
                <------------------------> <----------------------------------------------> 
                31                      22 21                                             0
                |___page_directory_index__|_____________________offset____________________|
                            |                                     |                       
                            |                                     |                      
            CR3 ----> +-----------+            +-----------> +----------+
               points |   Page    |            |             |   4 MB   |
                      | Directory |------------+             |   Page   |
                      +-----------+                          +----------+
            
            ```
       
    + Setting paritcular **flag in CR4 register**, this turns on **support for 4 MB pages**.

        ```asm
        #define CR4_PSE         0x00000010      // Page size extension
        
        # Entering xv6 on boot processor, with paging off.
        .globl entry
        entry:
            # Turn on page size extension for 4Mbyte pages
            movl    %cr4, %eax
            orl     $(CR4_PSE), %eax
            movl    %eax, %cr4
        ```

    + Setting **pointer to the page directory entries**(PDE) using CR3 register.
        
        ```asm
            # Set page directory
            movl    $(V2P_WO(entrypgdir)), %eax
            movl    %eax, %cr3
        ```

        + The **entrypgdir** is defined in **main.c** file, which is an **array
          of 1024 entries**, and each **entry is 4 bytes**, thus total 1MB or
          1 page memory reserved for the page directory for 1 MB pages.
         
            ```c
                
                #define NPDENTRIES      1024        // # directory entries per page directory
                #define KERNBASE        0x80000000  // First kernel virtual address
                
                // each entry of page directory is 4 bytes 
                typedef uint pde_t;

                // The boot page table used in entry.S and entryother.S.
                // Page directories (and page tables) must start on page boundaries,
                // hence the __aligned__ attribute.
                // PTE_PS in a page directory entry enables 4Mbyte pages.
                
                __attribute__((__aligned__(PGSIZE)))
                pde_t entrypgdir[NPDENTRIES] = {

                    // Map VA's [0, 4MB) to PA's [0, 4MB)
                    [0] = (0) | PTE_P | PTE_W | PTE_PS,

                    // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
                    [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
                };

            ```
            + **Question : where exactly is the location entrypgdir in memory ?**
                + The location of entrypddir can be obtained from **kernel.sym** 
                  file or by using objdump command. Virtual address is 0x80109000
            
            + **Note** : out of total 1024 entries in page directory, entrypddir
              array only 2 are initialized the **entry 0** and **entry 512**, and both 
              are mapped to page frame 0 in actual memory location with following 
              permissions bits set.
                + PTE_P  -> frame persent 
                + PTE_W  -> write permision on frame
                + PTE_PS -> 4 MB page frame
                  
    + Setting particular **flag in CR0 register to turn on paging**

        ```asm
            # Turn on paging.
            movl    %cr0, %eax
            orl     $(CR0_PG|CR0_WP), %eax
            movl    %eax, %cr0
        ```
    + **NOW FINALLY PAGING HAS STARTED**. Thus for every address issued the MMU
       will refer to the Page Directory Table pointed by CR3 regsiter.

    + **COOL STUFF : Every address issued is just mapped to the offset**

        + The two entries in page directory table are both mapping to the same 
          physical memory location that is frame 0. So effectively any address 
          issued by the CPU will get translated as offset iteself

        + Example address with 0 page number + offset = offset, but address with
          512 page number + offset ---> translated as ---> 0 + offset = offset

        + Currently **OFFSET == PHYSICAL ADDRESS**

* **JUMPING to MAIN kernel function**

    + Before jumping to main function we need to shift to the kernel stack.
      Note the bootloader while loading the kernel would have **loaded the kernel
      stack on the RAM** as it was **specified in the ELF file**.

    + The stack pointer is made to point to the memory location of kernel stack
        ```asm
            # Set up the stack pointer.
            movl $(stack + KSTACKSIZE), %esp
        ```
    + Finally jump to the main function 
        ```asm
            mov $main, %eax
            jmp *%eax
        ```

## MAIN FUNCTION (C code of kernel)

* The things done before this are, the bootloader had set up segmentation table 
  for MMU and loaded the kernel into RAM, the entry function had set up Page
  Directory table of 4 MB for MMU and finally passed control to main function.

    ```text
        start (in bootblock) -> bootmain() -> entry() -> main()
    ```

* The main function is in **main.c** file, which calls a lot of functions
    ```c
        // Bootstrap processor starts running C code here.
        // Allocate a real stack and switch to it, first
        // doing some setup required for memory allocator to work.

        int main(void) {
          kinit1(end, P2V(4*1024*1024)); // phys page allocator
          kvmalloc();      // kernel page table
          mpinit();        // detect other processors
          lapicinit();     // interrupt controller
          seginit();       // segment descriptors
          picinit();       // disable pic
          ioapicinit();    // another interrupt controller
          consoleinit();   // console hardware
          uartinit();      // serial port
          pinit();         // process table
          tvinit();        // trap vectors
          binit();         // buffer cache
          fileinit();      // file table
          ideinit();       // disk 
          startothers();   // start other processors
          kinit2(P2V(4*1024*1024), P2V(PHYSTOP)); // must come after startothers()
          userinit();      // first user process
          mpmain();        // finish this processor's setup
        }
    ```

