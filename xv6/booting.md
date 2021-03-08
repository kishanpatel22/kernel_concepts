# Bootstrapping the xv6 kernel 

## BootLoader of xv6 kernel

* The first code to get executed on the CPU after the power on is the BIOS,
  The BIOS then loads the BOOTLOADER at paritcular address (0x7C00) which then 
  in turn loads the xv6 kernel.
  
* The BOOTLOADER code(bootstrap process) for xv6 is written in the files 
  **bootasm.S** and **bootmain.c**. The make file first compiles the codes of
  bootloader with these files as show below
  
```sh
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock
```

* Some points on the compilation of the xv6 bootloader files
    + Both the files bootmain.c and bootasm.S are compiled using gcc and 
      then properly linked using the GNU linker 
    + The Flags uses in compilation and its importance 
        + **-N Flag** : Set the text and data sections to be readable and writable.
        + **-e Flag** : use entry as explicity symbol for beginning execution
                        of propram 
        + **-Ttext Flag** : Locate the text (code) section in the output file at the
                            absolute address given in arguments. Specifying 0x7C00. 
                            This tells us that the bootloader code will be
                            loaded at this paritcular memory location
    
    + The final result of compilation is getting the bootblock executable file 
      from the bootasm.S and bootmain.c

### 1) bootasm.S file 

* It's assembly language code which contains the bootstrap process code which
  in turn calls the bootmain.c function.

* It contains the **start** function which is first function which is executed
  as mentioned to the linker in the compilation. The start function contains
  the code which does the following work 

    + **Disable Interrupts** 

        + Why ? -> BIOS is like a tiny OS which could have set up its own
                   hardware interrupt codes as part of initializing the system.
                   Bootloader while loading kernel insures that no interrupts are serviced.

          ```asm
                cli                         # BIOS enabled interrupts; disable               
          ```

    + **Clear ax, ds, es, ss registers** 

        + Why ? -> BIOS doesn't guarantee anything about the register contents
                   so boot loaders responsibility to make them as zero.

          ```asm
                # Zero data segment registers DS, ES, and SS.                                
                xorw    %ax,%ax             # Set %ax to zero                                
                movw    %ax,%ds             # -> Data Segment                                
                movw    %ax,%es             # -> Extra Segment                               
                movw    %ax,%ss             # -> Stack Segment                               
          ```

    + **Enabling 21 bit address line**

        + Why ? -> It's compatibility hack to check if second bit of keyboard 
                   controller is low, this 21st address bit is always cleared 
                   or if high the 21st address bit acts normally. Intel 8088
                   and earlier processor had support to 20 bit address in logical 
                   format of **segment : offset** while later version had support
                   to more than 20 bit in the address format. In short there is 
                   I/O operation done with keyboard controller on ports 0x64 and 0x60.
 
          ```asm
                # Physical address line A20 is tied to zero so that the first PCs 
                # with 2 MB would run software that assumed 1 MB.  Undo that.
                seta20.1:
                      inb     $0x64,%al               # Wait for not busy
                      testb   $0x2,%al
                      jnz     seta20.1
                
                      movb    $0xd1,%al               # 0xd1 -> port 0x64
                      outb    %al,$0x64
                
                seta20.2:
                      inb     $0x64,%al               # Wait for not busy
                      testb   $0x2,%al
                      jnz     seta20.2
                
                      movb    $0xdf,%al               # 0xdf -> port 0x60
                      outb    %al,$0x60
          ```
         
    + **Real Mode to Protected mode**

        + **Real Mode** (16 bit)
            * CPU executes instruction with 16 bit address format, that is it
              makes use of the 8, 16 bit registers. Physical addresses are 
              20 bits, so 1 MB RAM. No protection thus program can load
              anything into segement registers. **xv6 Bootloarder starts in
              real mode but later convert to pretected mode**

        + **Protected Mode** (32 bit)
            * CPU executes instructions with 32 bit address format that is it 
              makes use of the 4, 32 bit registers. Physical address are of 32 
              bits, thus total of 4 GB (2 ^ 32) of memory becomes accessable 
        
        + These modes enables it **registers**, **virtual address** and most of 
          **integer arithematic** to be carried out in specified bits of mode.
        
        + Why need the above two modes ? -> this **ensures backward compatibility**,
          The earlier bios codes written can still be executed with the newer
          versions of the processors. Also the protection mode **allows the OS
          to set up address space so that user process can't change them**.
        
        + To change the mode from Real mode to Protected mode seperate set of 
          assembly language instructions are needed.

        + In protected mode **segment register index into a segment descriptor
          table**. Each table entry compries of 

          | Base address | maximum virtual address(limit) | permission/control bits |
          |--------------|--------------------------------|-------------------------|
          |     ...      |              ...               |         ...             |
                             
          What does the entry in each table do ? It translates the **logical
          address**(segment : offset) to linear address and this is called as 
          **Segment Translation**. 
        
        + **Segment Descripter Table is simply array in memory** which converts 
          the logical address into linear address using some arithematic inside CPU.

          <img src="https://pdos.csail.mit.edu/6.828/2005/readings/i386/fig5-2.gif" alt="ELF file" height="400px" widht="300px">

        + **Note the kernel is still not in the picture !**. The **bootloader is 
          setting up the table enabling the 32 bit protection mode** before kernel is
          loaded and executed. All the arithematic to convert the logical address 
          into linear address happens in side the CPU without any involement of kernel.

    + **xv6 Bootloader converting Real Mode to Protected mode**
        
        + The bootloader code executes an **lgdt** instruction to set the
          process **Global Descripter Table**(GDT) with a value of gdtdesc
          which points to the table gdt

          ```asm
                lgdt    gdtdesc
          ```

        + **Note that GDT in array in memory, CPU's GDTR register points to GDT**.
            As mentioned below gdtdesc is pointer to the GDT which is initialized
            by the lgdt instruction.
          ```asm
                # Bootstrap GDT
                .p2align 2                                # force 4 byte alignment
                gdt:
                    SEG_NULLASM                             # null seg
                    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
                    SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg
                
                gdtdesc:
                    .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
                    .long   gdt                             # 4 byte address of gdt
          ```

        + After this lgdt copies gdtdesc pointer into register as show below in file xv6.h
          ```asm
                static inline void lgdt(struct segdesc *p, int size) {
                    volatile ushort pd[3];
            
                    pd[0] = size-1;                         # size fo the table 
                    pd[1] = (uint)p;                        # lower 16 bits of address
                    pd[2] = (uint)p >> 16;                  # higher 16 bits of address
            
                    asm volatile("lgdt (%0)" : : "r" (pd));
                }
          ``` 
        + **Now finally change the protection bit by setting the PE bit in CR0 register**
            ```asm
                    // Control Register flags
                    #define CR0_PE          0x00000001      // Protection Enable Bit

                    movl    %cr0, %eax
                    orl     $CR0_PE, %eax               # set the protection bit 
                    movl    %eax, %cr0
            ```

        +  **What will the protection mode do ?**
            + instructions can only r/w/x memory reachable through segment registers
            + memory is inaccessible before base and similarly after limit.
            + how does hardware know if user or kernel?
                + Setting the current privilege level (CPL) which is the low 
                  2 bits of CS. CPL = 0 means privileged OS, CPL = 3 means user
                + To change mode from user to kernel mode just lower the CPL value.
  
    + **Long jump to the start32 function !** (Now switch process to 32 bit mode !)

        + Now since the **bootloader has changed to protection mode but actual
          switch to 32 bit mode happens when the code executes the long jump
          instruction to the start32 function** which will at end call the bootmain 
          function to load xv6 kernel.
    
        + After the long jmp the CPU contiunes to executes the next instructions but 
          **during the long jump the code segment register refers to entry in GDT 
          which decribes 32 bit code segment so processor switches to 32 bit mode**

            ```asm
                ljmp    $(SEG_KCODE<<3), $start32 
            ```

        +  **start32 function** will set the protected-mode data segment registers.

            ```asm
                start32:
                    # Set up the protected-mode data segment registers
                    movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
                    movw    %ax, %ds                # -> DS: Data Segment
                    movw    %ax, %es                # -> ES: Extra Segment
                    movw    %ax, %ss                # -> SS: Stack Segment
                    movw    $0, %ax                 # Zero segments not ready for use
                    movw    %ax, %fs                # -> FS
                    movw    %ax, %gs                # -> GS
            ```

        + Now finally setting up the stack pointer and call to bootmain function
          **Note the bootmain function should ideally never return, since it
            will finally hander over the CPU control to xv6 kernel**

            ```asm
                # Set up the stack pointer and call into C.
                movl    $start, %esp
                call    bootmain
            ```

### 2) bootmain.c file 

* **bootmain function** job is to load the kernel into the RAM and pass CPU control to kernel.

* The kernel is in EFL file format, so the bootmain has to read the kernel's
  ELF binary file format which is defined in the **elf.h** file. An ELF file 
  consists of **ELF header** which is followed by the **program section header**

    + The **ELF header structure** defined by **elfhdr** struct in the elf.h
      file. The struct defines the various sections in the ELF header.

        ```asm
            #define ELF_MAGIC 0x464C457FU  // "\x7FELF" in little endian
            
            // File header
            struct elfhdr {
                uint magic;  // must equal ELF_MAGIC
                uchar elf[12];
                ushort type;
                ushort machine;
                uint version;
                uint entry;
                uint phoff;
                uint shoff;
                uint flags;
                ushort ehsize;
                ushort phentsize;
                ushort phnum;
                ushort shentsize;
                ushort shnum;
                ushort shstrndx;
            };
        ```
        
        + The **ELF header points at a small number program headers** which
          describe the sections that make up the running image of the kernel


    + The **program section header structure** defined by **proghdr** struct in 
      elf.h. The struct defines the various sections in the program header
        
        ```asm
            // Program section header
            struct proghdr {
                uint type;
                uint off;
                uint vaddr;
                uint paddr;
                uint filesz;
                uint memsz;
                uint flags;
                uint align;
            }
        ```

        + Each program section header consits of location of the sections
          content **virtual address** (vaddr) relative to start of elf header 
          (offset), the **number of bytes to load from the file** (filesz) and 
          the **number of bytes to allocate memory** (memsz)

    + **Lets checkout the kernel file of the xv6**

        ```sh
            $ objdump -p kernel                     # prints the ELF program headers

                kernel:     file format elf32-i386
                Program Header:
                    LOAD off    0x00001000 vaddr 0x80100000 paddr 0x00100000 align 2**12
                         filesz 0x000078ac memsz 0x000078ac flags r-x
                    LOAD off    0x00009000 vaddr 0x80108000 paddr 0x00108000 align 2**12
                         filesz 0x00002516 memsz 0x0000d4a8 flags rw-
                   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
                         filesz 0x00000000 memsz 0x00000000 flags rwx
        ```
        
        + Notice that for the second section the file size is smaller that the
          allocated memory size, so remaining bytes in memory are set to zero.

        + The bootmain function is going to use these program headers to load
          the exact xv6 kernel code into the RAM memory.

    + **Now few doubts regarding loading the kernel**

        + **Question is how much memory of the kernel to load in RAM ?**

            + To read the ELF file headers, the bootloader's bootmain function 
              loads 4096 byes of the xv6 kernel (gross overestimation though,
              since the size must be multiple of 512 bytes or sector in harddisk)

            + **Bootloader must know exact size of the kernel that is to be loaded into RAM**

        + **Question is where to load the kernel inside the RAM ?**

            + Actaully bootloader can decide where to load the kerenel, however 
              the memory location should be 0x10000 and above.

            + why this ? -> reason is **below memory location 0x10000 the Bootloader
              code is present and is currently running while loading the OS and
              has still not handed over control to the kernel**. 
              
            + Typical memory map looks like as shown below, where **X = memory
              location where bootloader loads the kernel**
    
              ```shell
                         | Protected-mode kernel  |
                100000   +------------------------+
                         | I/O memory hole        |
                0A0000   +------------------------+
                         | Reserved for BIOS      | Leave as much as possible unused
                         ~                        ~
                         | Command line           | (Can also be below the X+10000 mark)
                X+10000  +------------------------+
                         | Stack/heap             | For use by the kernel real-mode code.
                X+08000  +------------------------+
                         | Kernel setup           | The kernel real-mode code.
                         | Kernel boot sector     | The kernel legacy boot sector.
                       X +------------------------+
                         | Boot loader            | <- Boot sector entry point 0x7C00
                001000   +------------------------+
                         | Reserved for MBR/BIOS  |
                000800   +------------------------+
                         | Typically used by MBR  |
                000600   +------------------------+
                         | BIOS use only          |
                000000   +------------------------+
              ```

    + **Now before loading the kernel code, bootmain functions does typecasting
      of the address from integers and structs in C langauge**

        + C language casts freely between pointers and integers, but for the
          undelying processor it makes no difference.

        + The start of the bootmain function 
            ```c
                /* address of RAM in which the xv6 kernel code will be loaded */
                struct elfhdr *elf = (struct elfhdr *)0x10000;
            ```
    + **Reading the 4096 bytes of xv6 kernel code from hardisk**

        ```c
            // Read 1st page off disk
            readseg((uchar*)elf, 4096, 0);
        ```
        + Note the function readseg, first parameter is the **address where
          data read from disk is stored**, second parameter is **bytes that are 
          to be read from disk**, and third parameter is **offset bytes relative 
          to kernel code**

   + **Bootloader checks that header of the ELF which is just now loaded**

        ```c
            // Is this an ELF executable?
            if(elf->magic != ELF_MAGIC)
                return;  // let bootasm.S handle error
        ```

    + **Bootloader loops reading the pointer to the program headers in the ELF**
        + This loads each code section with the helps of all the program headers
          read using the ELF header.
            ```c
                // Load each program segment (ignores ph flags).
                ph = (struct proghdr*)((uchar*)elf + elf->phoff);   /* pointers to program headers */
                eph = ph + elf->phnum;
                for(; ph < eph; ph++){
                    pa = (uchar*)ph->paddr;
                    readseg(pa, ph->filesz, ph->off);       /* reading particular section of kernel code */
                    if(ph->memsz > ph->filesz)
                        stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
                }
            ```

    + **Bootloader calling the kernels entry function** (Passing control to kernel) 

        + **How does the bootloader code gets to know where the main function of kernel is ?**

            + The elf header contains the address of the main function of the kernel.
              So in ELF file of kernel itself it's is mentioned that about main kernel function.

                ```c
                    // Call the entry point from the ELF header.
                    // Does not return!
                    entry = (void(*)(void))(elf->entry);
                    entry();
                ```


