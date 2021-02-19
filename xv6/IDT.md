# Interrupt Descripter Table 

* After the kernel getting the CPU control from the bootstrap process one of
  the main task is to set up the **Interrupt Descripter Table**(IDT) or also
  known as IVT.

* Why to do this ? -> Setting up this table will ensure that whenever any
  interrupt (hardware or software) occurs, its always the kernel code which
  gets executed.

## HANDLING TRAPS 

* Inorder for the kernel code to execute, **mode must be change** from user to
  kernel. Why ? -> the user code cannot run priviledge instruction which
  direcly deal with the underlying hardware, but kernel mode can !.

* There are only three situations where user mode is changed.
    + On a system call (which in turn will raise a software interrupt)
    + On a hardware interrupt.
    + On an execption occuring while CPU execution (example is divide by 0)

* In any of the situations as mentioned above, what happens is
    + switch the mode from user mode to kernel mode
    + CPU refers IVT and jumps to predefined location where kernel code(ISR) is written
    + The CPU refers the kernel stack for the subsequent operations.

* **Kernel stack Vs User stack ?**
    + Actually there kernel code can run making use of the user stack no
      such restriction, but this logical seperation ensures security.

* **Actions to be taken when the trap is generated !**
    + Save the context of the process in the PCB.
    + Set up the system to run kernel code (kernel context) on kernel stack.
    + Start the kernel in appropriate places
    + Kernel to get all the information related to the event.


## Privilege Levels 

* x86 processors support 4 protection levels or rings.
* In practice most of the OS use only two levels of privileges. 
    + **Kernel Mode** : level = 0
    + **User Mode** : level = 3
* The current priviledge level (CPL) is stored in CS register, the upper two bits.
* In case of INT, hardware interrupt and execption the CPL will change
  automatically and after interrupt service CPL will be restored using iret instruction.
* **Question : Thus how to convert from user mode to kernel mode ?**
    + simply lower the CPL

## Interrupt Descripter Table (IDT)

* **IDT defines interrupt handlers**
* IDT has 256 entries and each entry consists of **CS : IP** pair defining the
  predetermined location where the ISR code of the kernel is written.

* It is the job of the OS to the fill in the entries in the IDT, as soon as the
  bootloader hands over the control to the kernel.

* **Interrupts 0-31** are defined for software exceptions, like divide errors
  or attempts to access invalid memory address.

* **xv6** maps the 32 hardware interrupts to the range 32-63 and uses the
  **hardware interrupt 64 as system call interrupt**.

#### xv6 IDT 

* The **IDT structure** in xv6 is present in **mmu.h** file. Each entry of this
  structure is of 64 bits or 8 bytes. This structure is actually taken from
  Intels hardware manual pages.

```c
struct gatedesc {
    uint off_15_0 : 16;   // low 16 bits of offset in segment
    uint cs : 16;         // code segment selector
    uint args : 5;        // # args, 0 for interrupt/trap gates
    uint rsv1 : 3;        // reserved(should be zero I guess)
    uint type : 4;        // type(STS_{IG32,TG32})
    uint s : 1;           // must be 0 (system)
    uint dpl : 2;         // descriptor(meaning new) privilege level
    uint p : 1;           // Present
    uint off_31_16 : 16;  // high bits of offset in segment
};
```

* The kernel code present in **main.c** file, contains **main function** which
  gets called after the bootloader loads the kernel into the memory. Now one of
  calls from kernel's main function is to function named **tvinit** (trap
  vector init)

```c
    /* the sequence of funcion calls before tvinit */
    bootloader() -> bootmain() -> entry() -> main() -> tvinit()
```


* The idt is array containing 256 entries of the type **gatedesc** structure.
  It contains all the 256 **CS:IP** pairs, with some additional informations as
  show in the above structure. The initialization of each idt enty happens
  using the setgate macro as given below.
    
    + first parameter  : is **gatedesc entry to be initialized**
    + second parameter : is **indicator for trap or interrupt**
    + third parameter  : is **code segment selector(cs register)**
    + fourth parameter : is **vector containing the offset values(ip register)**
    + fifth parameter  : is **descripter priviledge level** 

```c

#define T_SYSCALL   64          // system call
#define SEG_KCODE   1           // kernel code
#define DPL_USER    0x3         // User DPL

struct gatedesc idt[256];       // Interrupt descriptor table (shared by all CPUs).
extern uint vectors[];          // in vectors.S: array of 256 entry pointers

void tvinit(void) {

    int i;
    for(i = 0; i < 256; i++) {
        SETGATE(idt[i], 0, SEG_KCODE<<3, vectors[i], 0);
    }
    SETGATE(idt[T_SYSCALL], 1, SEG_KCODE<<3, vectors[T_SYSCALL], DPL_USER);
}

```

* The idt array of 256 entries is initialized using the global vectors arrays. 
  The **vector array contains offset to memory location where OS code
  (ISR) is written**.


* **Question : where is the IDT array in memory ?**
    + The IDT array is global variable and it would have been allocated
      space in kernel ELF file generate after compliation. The linker would
      have linked all the locations where IDT array is used with appropriate
      address. The bootloader will futher load the kernel code in memory and 
      thus the IDT array will be at particular memory location. 


* **Note : the special entry of vector number 64 (T_SYSCALL)**
    + The entry is specially made since it indicates a system call entry and  
      has descripter priviledge level of user(which is 3). Bascially the entry
      64 in the idt array is reserved for the system calls.


* The **SETGATE is multiline macro** which is defined as given below. The
  parameters of the macro as defined as 
    + **gate**      : A gatedesc structure entry
    + **istrap**    : trap / interrupt indicator
    + **sel**       : code segment value
    + **off**       : instruction pointer offset
    + **d**         : descriptor privilege level

```c

#define STS_IG32    0xE     // 32-bit Interrupt Gate
#define STS_TG32    0xF     // 32-bit Trap Gate

// Set up a normal interrupt/trap gate descriptor.
// - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate.
//   interrupt gate clears FL_IF, trap gate leaves FL_IF alone
// - sel: Code segment selector for interrupt/trap handler
// - off: Offset in code segment for interrupt/trap handler
// - dpl: Descriptor Privilege Level -
//        the privilege level required for software to invoke
//        this interrupt/trap gate explicitly using an int instruction.

#define SETGATE(gate, istrap, sel, off, d)                  \
{                                                           \
  (gate).off_15_0 = (uint)(off) & 0xffff;                   \
  (gate).cs = (sel);                                        \
  (gate).args = 0;                                          \
  (gate).rsv1 = 0;                                          \
  (gate).type = (istrap) ? STS_TG32 : STS_IG32;             \
  (gate).s = 0;                                             \
  (gate).dpl = (d);                                         \
  (gate).p = 1;                                             \
  (gate).off_31_16 = (uint)(off) >> 16;                     \  
}

```

* The **vector array** is external global variable, which is declared in the
  **vectors.S** file. Each **entry in array is address to given by vector label**
   
```asm
    # vector table
    .data
    .globl vectors
    vectors:
        .long vector0
        .long vector1
        .long vector3
        ...
        ...
        ...
        .long vector255
```

* The vector label in xv6 is defined as given below. Note that at **this address 
  OS code is not stored rather some other assembly code is written which can take
  to the location where the OS ISR is written**.
    ```asm
    
    .globl vector0
    vector0:
        pushl $0
        pushl $0
        jmp alltraps

    .globl vector1
    vector1:
        pushl $0
        pushl $1
        jmp alltraps
    
    ...
    ...
    ...

    .globl vector255
    vector1:
        pushl $0
        pushl $255
        jmp alltraps

    ```
    +  The location specified by any vector entry contains code which pushes
       first zero and then pushes the interrupt number on the stack and jmps to
       the fuction called alltraps.

* The assembly code written by each vector label is very similar, just pushing
  two values, and the jumping to the **alltraps routine**, which is written in
  the **trapasm.S** file.

    ```asm
        .globl alltraps
        alltraps:
            # Build trap frame.
            pushl %ds
            pushl %es
            pushl %fs
            pushl %gs
            # push all the general purpose register on stack 
            pushal
            
            # Set up data segments.
            movw $(SEG_KDATA<<3), %ax
            movw %ax, %ds
            movw %ax, %es
    
            # Call trap(tf), where tf=%esp
            pushl %esp
            call trap
            addl $4, %esp
            
        # Return falls through to trapret...
        .globl trapret
        trapret:
              popal
              popl %gs
              popl %fs
              popl %es
              popl %ds
              addl $0x8, %esp  # trapno and errcode
              iret
    ```

    + There are few push instructions first, then the data segement and extra
      segment are set with respect to the kernel address space, and finally
      after pushing the stack pointer, call to **trap function** happens.
    + **Question : Why push these registers on stack before calling trap function ?**
        + Reason 1 : Remember the calling convention for the functions the parameters are
                     passed by stack to the callee
        + Reason 2 : Kernel trap function gets to know the context of the given process


* **Trap** is the function responsible of handling all the interrupts occuring



* Still important work is left, even after seting up the IDT table with the
  appropriate entries of ISR. The work is **initializing register in CPU called IDTR 
  to point to the location where the IDT is residing in memory**. 

    + The **main function** of the kernel at last call the **mpmain** function
      which futher calls the **idinit** function which further calls the **lidt**
      function which initializes the CPU register with the location of IDT.
        
        ```c
            /* sequence of function calls
             * main() -> mpmain() -> idtinit() -> lidt()
             */
            static inline void lidt(struct gatedesc *p, int size) {

              volatile ushort pd[3];
            
              pd[0] = size-1;
              pd[1] = (uint)p;
              pd[2] = (uint)p >> 16;
            
              asm volatile("lidt (%0)" : : "r" (pd));
            }
        ```

* **Now finally the IDT is setup !!!**

