### SEGINIT FUNCTION (UPDATING THE GDT)

* The segmentation which is was set up for xv6 before loading the kernel into
  RAM, which is effectively bypassed using the entries in Global Descripter Table.
  Now there is again initialization of the segmentation table done by the kernel.

    ```c
        seginit();       // segment descriptors
    ```

* **seginit()** function updates the global descripter table (GDT) by adding 
  two more additional values in the table, which corresponds to the process
  address space segments. 
    
    ```c
        #define SEG_KCODE 1  // kernel code
        #define SEG_KDATA 2  // kernel data+stack
        #define SEG_UCODE 3  // user code
        #define SEG_UDATA 4  // user data+stack

        #define DPL_USER    0x3     // User DPL

        // Set up CPU's kernel segment descriptors.
        // Run once on entry on each CPU.

        void seginit(void) {

          struct cpu *c;
        
          // Map "logical" addresses to virtual addresses using identity map.
          // Cannot share a CODE descriptor for both kernel and user
          // because it would have to have DPL_USR, but the CPU forbids
          // an interrupt from CPL=0 to DPL=3.
          c = &cpus[cpuid()];

          c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
          c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
          c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
          c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);

          /* update the GDT resgister pointed memory location */
          lgdt(c->gdt, sizeof(c->gdt));
        }
    ```
    + In total, till now **5 segmentation entry have been setup**. The first
      entry is null and rest four are defined above.
     
    + GDT register is maintained per CPU, thus there is function call to
      **cpuid()** which returns the current CPU index in the CPU array. Thus
      the segmentation for each CPU would different.
    
    + **Each CPU core has seperate GDT table, and thus might have different GDT entries.**

    + **The entries current are setup for only one CPU (uniprocessor system)**.
      Note the **startothers()** function setups the GDT table for multiprocessor system.

    + Note the only difference between the first two and last two values is
      about the permissions. 
        + **User processes** for it's execution refer the last two values in GDT.
           Thus there have lower priviledge level.
        + **Kernel** for it's execution refer the first two values in GDT 
           Thus there have Higher priviledge level.
      
    + However for user processes or kernel code execution the logical address
      and linear address both are same, segmentation is still effectively
      bypassed for both.
       
    + **From now GDT is never changed !!**. The page tables setup can still be
      updated with new entires but not the GDT.



