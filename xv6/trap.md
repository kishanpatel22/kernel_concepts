# HANDLING INTERRUPTS 

* **INT** instruction will generate a software interrupt, similar in case of
  the hardware interrupt occuring on the **INTR pin of the CPU**, some specific 
  actions need to be taken.

* **Question : Atomicity in case Hardware interrupts ?**
    + Hardware interrupts being asynchronous, will not have any affect the execution 
      of the current instruction on the CPU. Only after the completion of the
      instruction the service to that interrupt will take place.
    + **Thus interrupts only occur between two assembly instructions**. 
      This will ensure atomicity of the process context.

* All the interrupts are handled by a single function **trap**, which is called
  with certain parameters oftenly referred to as **trapframe**.

* **Note : trap is handled in the kernel stack and not in the current process stack**.
  We need first to change the mode from user mode to kernel mode and need some
  mechanism to change the stack pointer to point to the kernel stack.

#### SWITCHING TO KERNEL STACK 

* In case of interrupt, first thing is to change the privilege level change 
  from user to kernel mode, it also switches to a stack in the kernel's memory. 

* **Task State Segment** : It's register which contains the pointer to the 
  Global Descripter Table (GDT), which contains entries of the segment
  selector and offset indicating where the kernel stack is in memory.

  ```text
                   points              points 
    TSS register ---------> GDT entry ---------> Kernel stack location
  ```

## Executing the INT instruction 

* **Fetch the nth descripter from the IDT** where n the argument to INT instruction
  or the interrupt number recieved when the INTR pin becomes high
    + **Question : Where is IDT located?** -> the location is stored by kernel in
      IDTR register as part of the init process(remember idtinit() function call).

* Check if the Current Priviledge Level (CPL) <= Discripter Priviledge Level (DPL).
  Do this in case of INT instruction only !
    + **Question : where is CPL and DPL?** -> CPL will the current CS regsiter
      upper two bits and DPL will be entry in the nth descripter fetched from IDT.

* **Switching to kernel stack !**. Save the SP and SS register in CPU internal
  registers if the target segment priviledge level <= CPL. Now load the SP and SS
  of the kernel stack.
    + **Question : where is kernel SP and SS** -> remember the TSS register
    + Now we are finally pointing to the kernel stack.

* **In the kernel stack push the registers**
    + push the process context SS
    + push the process context SP
    + push the process context FLAGS       
    + push the process context CS          
    + push the process context IP         
    
    ```text
                     +---------------------+ &cpu->kstackhi
                     |  proc context SS    |     " - 4
                     |  proc context SP    |     " - 8
                     |  proc context FLAGS |     " - 12
                     |  proc context CS    |     " - 16
                     |  proc context IP    |     " - 20 
                     +---------------------+  <---- ESP
    ```

    + Pushing the CP and IP, is common thing to do in case of far function call, 
      INT instruction is special in which flags are also pushed.
    + **Question : why push these resgisters** -> when process resumes from 
      interrupt the context must be saved, also the kernel gets to know the
      current context of the process in which interrupt has occured.

* **Clear EFLAG bits and Load the CS, IP with values specified in nth IDT which
  is already fetched**
    + **Question : what is there is the CP, IP of IDT ?** -> when kernel was
      setting up the IDT, remember we had array of 256 vector label entries,
      and IDT entries contains the CP, IP of the paritcular vector label.
    + **Note the function call sequence !**
        ```text
                  int call                                       jmp                   call
            int n --------> nth vector label (entry in IDT) ----------> alltraps() -----------> traps()
                  <----------------------------------------------------            <-----------
                                        iret                                          ret
        ``` 

#### Kernel code Execution for the interrupt

* **Control transfered to vector label**

    + In vector label as we know that only two things are pushed, the error
      code and the interrupt number. 

        ```asm
            vector1:
                pushl $0
                pushl $1
                jmp alltraps
            ...
            ...
        ```

    + The call to vector label is special function call, since it is made by
      INT instruction, contains CS + IP + FLAGS on the stack.

    + The current contents in the kernel stack 
        ```text
                     +----------------------+ &cpu->kstackhi
                     |  proc context   SS   |     " - 4
                     |  proc context   SP   |     " - 8
                     |  proc context FLAGS  |     " - 12
                     |  proc context   CS   |     " - 16
                     |  proc context   IP   |     " - 20 
                     |     error code(0)    |     " - 24 
                     |   interrupt number   |     " - 28 
                     +----------------------+  <----- SP
        ```
    
* **Jump(not call !) to alltraps subroutine**

    + The alltraps function actually build the left over trap frame, by pushing
      the general purpose registers on the stack, and call the generic trap
      function where the kernel code handles all the interrupts.
    
    + Push the following registers 
        + push process context ds
        + push process context es
        + push process context fs
        + push process context gs
        + push process context general purpose registers (ax, cx, dx, bx, osp, bp, si, di)
                     
    + The current contents in the kernel stack 
        ```text
                     +----------------------+ &cpu->kstackhi
                     |  proc context   SS   |     " - 4
                     |  proc context   SP   |     " - 8
                     |  proc context FLAGS  |     " - 12
                     |  proc context   CS   |     " - 16
                     |  proc context   IP   |     " - 20 
                     |     error code(0)    |     " - 24 
                     |   interrupt number   |     " - 28 
                     |  proc context   DS   |     " - 32
                     |  proc context   ES   |     " - 36 
                     |  proc context   FS   |     " - 40 
                     |  proc context   GS   |     " - 44
                     |  proc context   AX   |     " - 48 
                     |  proc context   CX   |     " - 52
                     |  proc context   DX   |     " - 56 
                     |  proc context   BX   |     " - 60
                     |  kernel context OSP  |     " - 64 
                     |  proc context   BP   |     " - 68
                     |  proc context   SI   |     " - 72 
                     |  proc context   DI   |     " - 76
                     +----------------------+  <----- SP
        ```
    
    + **Two stack pointers on trap frame !!**
        + The stack pointer SP in the context of process, that is paritcular
          process which raised the interrupt, had it's SP saved on kernel stack.
        + The stack pointer OSP was saved by kernel as result of **pushal**
          assembly instruction, although saving this pointer is of no use.
    
    + **Now all process context is saved on the kernel stack**


* **Call to the trap function !!**

    + **trap** is a generic function which will handle all the interrupts
      occuring in the system.

    + **Note** : before call to trap the DS, and ES register are modified to
      point to kernel data segment and extra segment memory locations.
        + **Question : can we modify these register ? and why to modify**
            + Yes ! offcourse now we can modify these register since the
              context of the whole process is saved.
            + The trap function will required kernel data and extra segment for
              its execution, cannot use segments of that of the process, since
              (kernel address space != user address space)

        ```asm
            # Set up data segments.
            movw $(SEG_KDATA<<3), %ax
            movw %ax, %ds
            movw %ax, %es
    
            # Call trap(tf), where tf=%esp
            pushl %esp
            call trap
        ```
    
    + Now the current contents of the stack frame. (trap function IP is
      function since it's call instruction not jmp)

        ```text
        
        int instuction---> +----------------------+ &cpu->kstackhi
        special call       |  proc context   SS   |     " - 4
                           |  proc context   SP   |     " - 8
                           |  proc context FLAGS  |     " - 12
                           |  proc context   CS   |     " - 16
                           |  proc context   IP   |     " - 20 
        vector code   ---> |     error code(0)    |     " - 24 
                           |   interrupt number   |     " - 28 
          jmp to      ---> |  proc context   DS   |     " - 32
         alltraps          |  proc context   ES   |     " - 36 
                           |  proc context   FS   |     " - 40 
                           |  proc context   GS   |     " - 44
                           |  proc context   AX   |     " - 48 
                           |  proc context   CX   |     " - 52
                           |  proc context   DX   |     " - 56 
                           |  proc context   BX   |     " - 60
                           |  kernel context OSP  |     " - 64 
                           |  proc context   BP   |     " - 68
                           |  proc context   SI   |     " - 72 
                           |  proc context   DI   |     " - 76
        call to trap  ---> |  kernel context OIP  |     " - 80
                           +----------------------+  <----- SP
        ```
   
    + **One million dollar question : how does the trap function gets to the
       know the which interrupt occured and from which process context ?**
        + Answer -> **trap frame which is created on the kernel stack**, 
          it represents both the context of the process as well the interrupt
          which has been raised.
            
    + The trap function prototype is something like this

        ```c
            void trap(struct trapframe *tf);
        ```

        + Note an assembly instrucion call the trap function, thus the parameters 
          passed by the kernel stack, which are nothing but the trap frame.

    + The trapframe structure basically contains all the parameters but in the
       reverse order as pushed on to the kernel stack.

        ```c
            struct trapframe {
                // registers as pushed by pusha
                uint edi;
                uint esi;
                uint ebp;
                uint oesp;      // useless & ignored
                uint ebx;
                uint edx;
                uint ecx;
                uint eax;

                // rest of trap frame
                ushort gs;
                ushort padding1;
                ushort fs;
                ushort padding2;
                ushort es;
                ushort padding3;
                ushort ds;
                ushort padding4;
                uint trapno;

                // below here defined by x86 hardware
                uint err;
                uint eip;
                ushort cs;
                ushort padding5;
                uint eflags;

                // below here only when crossing rings, such as from user to kernel
                uint esp;
                ushort ss;
                ushort padding6;
            };
        ```
    + Using the trapframe structure kernel basically traps the process context 
      and type of interrupt, occuring at any given time.
        
* **Returing to the process and restoring the process context !!**  
    + Return calls are very interesting !
    + Trap returns just like a normal function (removing local variables poping 
      the IP, jumping to the IP)
    + Return from alltraps is very special (remember alltraps, was never
      called, it was jump instruction which lead to alltraps from vector label)
        + **alltrap restores the process context from the trapframe** : pops
          all the general purpose registers back.
            ```asm
                popal
                popl %gs
                popl %fs
                popl %es
                popl %ds
            ```

        + The current trap frame
            ```text
                     +----------------------+ &cpu->kstackhi
                     |  proc context   SS   |     " - 4
                     |  proc context   SP   |     " - 8
                     |  proc context FLAGS  |     " - 12
                     |  proc context   CS   |     " - 16
                     |  proc context   IP   |     " - 20 
                     |     error code(0)    |     " - 24 
                     |   interrupt number   |     " - 28 
                     +----------------------+  <----- SP
            ```

        + **alltrap does a smart thing** change SP that is increments it by 8
          bytes, this escapes the stack frame created by vector label. And then 
          execute iret instruction, which restores the flags, CS and IP.
            
            + Escaping the trap frame created by vector label functions
                ```asm
                    addl $0x8, %esp  # trapno and errcode
                ```

            + The current trap frame
                 ```text
                     +----------------------+ &cpu->kstackhi
                     |  proc context   SS   |     " - 4
                     |  proc context   SP   |     " - 8
                     |  proc context FLAGS  |     " - 12
                     |  proc context   CS   |     " - 16
                     |  proc context   IP   |     " - 20 
                     +----------------------+  <----- SP
                ```

            + The special return instruction 
                ```asm
                    iret
                ```
            
            + The current trap frame
                ```text
                         +----------------------+ &cpu->kstackhi
                         |  proc context   SS   |     " - 4
                         |  proc context   SP   |     " - 8
                         +----------------------+  <----- SP
                ```

            + **iret** is very special instruction does returns work for int 
              instruction call
            +  int instruction call **pushing flags, CS and IP** and changes mode 
               of **user mode to kernel mode**, similarly iret function returns
               by **poping IP, CS and flags** and thus changing mode from **kernel
               mode to user mode**

## HAPPY INTERRUPTING !!

