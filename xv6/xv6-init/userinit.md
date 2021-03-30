## USERINIT : creating the first processes for kernel.

* The kernel creates the very first process named as **init** by hand. The rest
  processes create would be forked from the init process. 
  
*  **Handcrafting the code** for init process is necessary since till now there no
   concept of processes created by the kernel i.e. **no one there to fork from**.

### INIT PROCESS creation 

* Remember the **fork-exec** codes which help in creating new child processes.
  The need some similar mechanism here, but again there is nothing to fork.
 
* Now we create very first process on xv6 kernel by handcrafted code, using the
  same **fork-exec** code, but before this xv6 creates processes as if it was 
  created by fork.

* Ensure that the processes returns and starts in the call to **exec**.
  In this case exec calls looks something this 

    ```c
        exec("/init", NULL);
    ```
    + Note but before exec must be called only when process is exits, forked
      and is running.

* Rememeber init is userland code written in **init.c** and **initcode.S** file. 

### INITCODE and INIT (what exactly the incode code does)

* Before even reading code let's see how the makefile complies the initcode.S

    ```sh
        initcode: initcode.S
        $(CC) $(CFLAGS) -nostdinc -I. -c initcode.S
        $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o initcode.out initcode.o
        $(OBJCOPY) -S -O binary initcode.out initcode
        $(OBJDUMP) -S initcode.o > initcode.asm
    ```

    * The flags used during compilation are -
        + **-nostdinc** : flag tells the compiler no to search for standard header files
        + **-I** : flags indicating the user include directory 
        + **-e** : specify the entry point for the execution.
        + **-Ttext** : specify the starting address explicitly

    * Note further the initcode excutable file is linked with the kernel.

* **Question - Why two files initcode.S and init.c?**
    + The trick here is initcode.S actually calls the init.c, so the exec 
      of **init** named executable file, leads the control to initcode.S and
      then to init.c

* The initcode.S file defined the start function which is the function which 
  will be called first as part init process.

    ```asm
        # exec(init, argv)
        .globl start
        start:
            pushl $argv
            pushl $init
            pushl $0  // where caller pc would be
            movl $SYS_exec, %eax
            int $T_SYSCALL
    ```asm



* The first thing the function of init does is it **opens file** named 
  **console** in READ and WRITE mode. 

    ```c
    char *argv[] = { "sh", 0 };
    
    int main(void) {
        int pid, wpid;
    
        if(open("console", O_RDWR) < 0){
          mknod("console", 1, 1);
          open("console", O_RDWR);
        }
    ```

* Now **dup** system call, initializes the stdin and stdout file descripter 
  for the init process.

    ```c
    dup(0);  // stdout
    dup(0);  // stderr
    ```

* Now next the init process calls fork to shell processes which is basically 
  the prompt that you see when you start run xv6 code.

    ```c
    for(;;) {
        
        /* fork-exec for the shell process */
        printf(1, "init: starting sh\n");
        pid = fork();
        if(pid < 0) {
          printf(1, "init: fork failed\n");
          exit();
        }
        
        /* child procees calls exec and starts the shell procees */ 
        if(pid == 0) {
          exec("sh", argv);
          printf(1, "init: exec sh failed\n");
                exit();
        }
    ```

* The init which is parent process keeps on wait for zombie proceess. 
    + **What is zombie process** ? -> the processes which doesn't have any parent.
      The parent must have terminated before even the child get over, thus
      parent doesn't call wait.

* For zombie processes, the init function takes care of removing the PCB and 
  handling the termination of child.

    ```c
        while((wpid=wait()) >= 0 && wpid != pid) 
            printf(1, "zombie!\n");
        }
    }
    ```

### USERINIT kernel function (create a feel of fork for init process)

* **userinit** function is called by the main function of the kernel, which was
  is invoked after bootblock had loaded the kernel. The userint function is
  written in **proc.c** file.

    ```c
        int main(void) {
            /* ... */     
            userinit();      // first user process
            /* ... */     
        }
    ```

* The **userinit** functions starts with **allocproc call**, which allocate 
  process control block(PCB) structure 
    ```c
        void userinit(void) {
          struct proc *p;
          p = allocproc();
    ``` 
    + However not allocproc not allocates structure for PCB but also initializes 
      the allocated PCB.

        + First allocproc **finds a UNUSED process state structure** from array 
          of process structure named **ptable**. After finding the structure 
          the **state is changed from UNUSED to EMBRYO** and pid is assigned.

            ```c 
                static struct proc* allocproc(void) {
                    struct proc *p;
                    char *sp;

                    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
                        if(p->state == UNUSED)
                            goto found;

                    return 0;

                    found:
                        p->state = EMBRYO;
                        p->pid = nextpid++;
            ```
        
            ```text
                         structure proc 
               p--> +-----------------------+
                    |    state = EMBRYO     |
                    |      pid = nextpid    |
                    +-----------------------+
            ```

        + Next we allocate the kernel stack to the process. Note the **kernel
          stack occupies space of PAGESIZE** which is 4096 for xv6.

            + **Question why allocate kernel stack ?**  -> simply on software 
              interrupts, system calls or hardware interrupts kernel runs on
              behalf of the process thus it needs a seperate stack for execution.

            ```c
                // Allocate kernel stack.
                if((p->kstack = kalloc()) == 0){
                    p->state = UNUSED;
                    return 0;
                }
            ```
            
            ```text                                    
                                                       +------------+   + (higher address)
                                                       |            |   |
                         structure proc                |            |   |
               p--> +-----------------------+          |            |   |
                    |    state = EMBRYO     |          |            |   | 
                    |      pid = nextpid    |          |            |   | PAGESIZE
                    |        kstack         |-----+    |            |   | 
                    +-----------------------+     |    |            |   |
                                                  |    |            |   |
                                                  |    |            |   |
                                                  +--> +------------+   +  (lower address)
            ```

        + Now remember allocproc's work is initialize every field of proc structure.
          Thus the initialization of the kernel stack memory.
            
        + Take pointer point to the base of the kernel stack which is basically 
          the end of the page which is allocated to the process. **Decrement the 
          stack pointer by size of trap frame** and **initialize the trapframe point to 
          the stack pointer**.

            ```c
                #define KSTACKSIZE 4096  // size of per-process kernel stack
                
                char *sp;
                sp = p->kstack + KSTACKSIZE;
                
                // Leave room for trap frame.
                sp -= sizeof *p->tf;
                p->tf = (struct trapframe*)sp;
            ```

            ```text                                    
                                                        +------------+   + (higher address)
                                                        |   sizeof   |   |
                         structure proc                 |  trapframe |   |
               p--> +-----------------------+     +---> |------------|   |
                    |    state = EMBRYO     |     |     |            |   | 
                    |      pid = nextpid    |     |     |            |   | 
                    |      trapframe tf     |-----+     |            |   | PAGESIZE
                    |        kstack         |-----+     |            |   | 
                    +-----------------------+     |     |            |   |
                                                  |     |            |   |
                                                  |     |            |   |
                                                  +---> +------------+   + (lower address)
            ```
 
        + Now allocproc setup the return address from the trap call, which is 
          **trapret**, this is discussed in trap.md file. Basically it's
          assembly langauge code which helps in returning from trap, 
          hardware interrupt or int instruction.  

            ```c
                // Set up new context to start executing at forkret which returns to trapret.
                sp -= 4;
                *(uint*)sp = (uint)trapret;
            ```

            ```text                                    
                                                        +---------------+   + (higher address)
                                                        |     sizeof    |   |
                         structure proc                 |    trapframe  |   |
               p--> +-----------------------+   +-----> |---------------|   |
                    |    state = EMBRYO     |   |       |  addr trapret |   | 
                    |      pid = nextpid    |   |  sp-> |---------------|   | 
                    |      trapframe tf     |---+       |               |   | PAGESIZE
                    |        kstack         |-----+     |               |   | 
                    +-----------------------+     |     |               |   |
                                                  |     |               |   |
                                                  |     |               |   |
                                                  +---> +---------------+   + (lower address)
            ```

        + Now allocproc initializes the context of the process. The **stack
          pointer is decremented by sizeof context** and context is initialized
          with all values as 0. However **eip is explicitly set to forkret**.

            ```c
                sp -= sizeof *p->context;                       
                p->context = (struct context*)sp;               /* setting up context */
                memset(p->context, 0, sizeof *p->context);      /* initializing context */
                p->context->eip = (uint)forkret;                /* initializing eip */
            ```

            ```text                                    
                                                        +---------------+   + (higher address)
                                                        |     sizeof    |   |
                         structure proc                 |    trapframe  |   |
               p--> +-----------------------+   +-----> |---------------|   |
                    |    state = EMBRYO     |   |       |  addr trapret |   | 
                    |      pid = nextpid    |   |       |---------------|   | 
                    |      trapframe tf     |---+       | eip = forkret |   | PAGESIZE
                    |       context         |-------+   |      ebp      |   |
                    |        kstack         |-----+ |   |      ebx      |   | 
                    +-----------------------+     | |   |      esi      |   |
                                                  | |   |      edi      |   |
                                                  | +-> |---------------|   |
                                                  |     |               |   |
                                                  |     |               |   |
                                                  |     |               |   |
                                                  +---> +---------------+   + (lower address)
            ```

            + However remember the forkret happens when the process returns 
              from fork. Thus when the context of the function gets poped the 
              control is transfered to the forkret function.
            
            + Remember when the context gets poped in shedular's switch
              function, thus by initialization the control will be transfered 
              to the forkret function.

* The **userinit** functions after getting and initializing new process structure,
  setups the **kernel page tables** and **user page tables**. Note that memory
  address for allocated to **kernel page tables is above KERNBASE** and **user
  process page tables is below KERNBASE**
    
    + The kernel pages are setup using the setkvm function
        ```c
            p->pgdir = setupkvm())
        ```
    
    + The user pages are setup using the inituvm function
        ```c
            inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
        ```
        
        + The inituvm function is similar to setupkvm initializes the size of 
          the init process to one pagesize. It **allocates and initializes one
          page to the init process and updates the pgdir**.
         
        + The variables **_binary_initcode_start** and **_binary_initcode_size**
          are extern variables.

            ```c
                void inituvm(pde_t *pgdir, char *init, uint sz) {
                    
                    char *mem;
                    if(sz >= PGSIZE)
                        panic("inituvm: more than a page");

                    /* Only a single page memory is allocated to the init process */
                    mem = kalloc();
                    memset(mem, 0, PGSIZE);
                    
                    /* the pagaes for init process are maped to 0th virtual address address space */
                    mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W|PTE_U);
                    memmove(mem, init, sz);
                }
            ```

        + **Question : whay extacly does memove copy to the memory allocated by the kernel ?**

            + The argument passed to inituvm was **_binary_initcode_start** is
              every variable which is set up by the linker after compilation,
              it's basically **start function of initcode.S** file. Note the 
              **initcode is part of the function**
    
        + **Note : size of first process i.e. init process is set to 1 KB**

            ```text
                                                                            +---------------+   + (higher address)
                                                                            |     sizeof    |   |
                                             structure proc                 |    trapframe  |   |
                                   p--> +-----------------------+   +-----> |---------------|   |
                                        |    state = EMBRYO     |   |       |  addr trapret |   | 
                                        |      pid = nextpid    |   |       |---------------|   | 
                                        |      trapframe tf     |---+       | eip = forkret |   | PAGESIZE
                                        |       context         |-------+   |      ebp      |   |
                                        |        kstack         |-----+ |   |      ebx      |   | 
                                        |         pgdir         |--+  | |   |      esi      |   |
                                        +-----------------------+  |  | |   |      edi      |   |
                                                                   |  | +-> |---------------|   |
                  +-----------+                                    |  |     |               |   |
                  | text/code |<---- +-------+ <-----+             |  |     |               |   |
                  +-----------+      |       |       |             |  |     |               |   |
                              <----- +-------+       |             |  |     |               |   |
                              page table entries     |             |  |     |               |   |
                                                     |             |  |     |               |   |
              +--------------+ <---- +-------+ <---+ |             |  +---> +---------------+   + (lower address)
              | kernel pages |       |       |     | |             |
              +--------------+       +-------+     | |             | 
                               page table entries  | |             |                               
                                      ....         | |             |                               
                                      ....         | |             |                               
                                      ....         | |             |  
                              <--- +-------+ <---+ | |             |                                
                              <--- |       |     | | |  +-------+  |                               
                                   +-------+     | | +--|       |  |                                
                             page table entries  | +----|       |  |
                                                 +------|       |  |
                                                        +-------+ <+ 
                                                page directory table entries
            ```

* **Note : size of first process i.e. init process is set to 1 KB**. The
  process code is defined in initcode.S named **start**. It is assembly
  langauge code which exec system call.

    ```c
        p->sz = PGSIZE;
    ```

* **Initialization** of the traframe of the process. All the general purpose  
  registers  are initialized as given.

    ```c    
        memset(p->tf, 0, sizeof(*p->tf));
        p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
        p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
        p->tf->es = p->tf->ds;
        p->tf->ss = p->tf->ds;
        p->tf->eflags = FL_IF;
        p->tf->esp = PGSIZE;        // stack pointer 
        p->tf->eip = 0;             // beginning of initcode.S
    ```
    
    + **Question the user process stack pointer is set to PGSIZE ?**

        + The code in initcode.S is very small starting from addrees 0, so 
          what xv6 does it manipulates the user stack allowing to grow
          downward from 4 KB which is the PGSIZE. Remember size of whole init
          process as such is 4KB.

* **Set up the name, inode (current working directory) and state of the process
  init process in the proc structure**. 
    
    ```c
            safestrcpy(p->name, "initcode", sizeof(p->name));
            p->cwd = namei("/");
            p->state = RUNNABLE;

            return;
        }
    ```

* Thats it we have actuall created a process which seems similar to what forked
  would have initializing all the data member of proc structure. **Question :
  when does the init process execute ?**


## MPMAIN : calling the schedular explicitly for the init process 
   
* Now for the first init process to run xv6 explicitly calls the schedular code.
    
    + First there is **initialization of the idt table**, simiply making the
      idt register pointer to the IDT table which is all ready setup by **tvinit**
    
    + Now there is **call to schedular**. Note mpmain only function which calls
      schedular explicitly apart from the coroutine sched. Thus no function 
      directly calls schedular execpt the mpmain.

    ```c
        // Common CPU setup code.
        static void mpmain(void) {
            idtinit();        // load idt register
            scheduler();      // start running processes
        }
    ```
* **Climax in the story**  

    + The schedular now **first enables interrupts**. Remember from
      bootloader the interrupts were disabled. **Now xv6 is in asynchrouns
      world, any type interrupt can occur.**

    + **Some critical points to be noted before the schedular doing the task**
        + The **esp** is still pointing to the stack which is allocated in **entry.S**.
          It is kernel stack, not per-process stack.
        
        + The **cr3** register is still pointing to kernel page directory **kgpgdir**.
    
    + Before call to switching the context **switchkvm** does works of setting
      up the stack pointer and cr3 register.

    + The schedular now simply traverses the the array of proc structures and
      **schedular finds the only process RUNNABLE to be as the init process**
      and runs the code is init process.

    + **The schedular does context switch**. Now **Switch returns back to
      forkret and not to init process**. And **forkret is kernel function
      which does some initialization and returns control to init processs.**
        
    + **Remember finally control transfered to start function of initcode.S**

* **Sequence of the calls and switches are defined as follow** 

    ```text
        
        /* initializes the first PCB */
        main() -> userinit()        
                                                                            (exec system call)
        main() -> mpmain() -> schedular() -> switch -> forkret() -> initcode -----------------> init
    ```
