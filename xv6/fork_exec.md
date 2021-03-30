# fork() in xv6 

## What must fork do ?

* Create a copy of exsisting process 
* Child is same as parent, except pid, child-parent relationship, return value.
* Create **new struct proc** 

    |   Type of copying |           Entries in new child proc               |
    |-------------------|---------------------------------------------------|
    |   Deep copy       |            pages, page directory, kstack          |
    |  Shallow copy     |              sz, context, ofile, cwd              |
    
    + However some values cannot be copied as it is they would be modified 
      **pid, parent pointer, trapframe and state**.
       
## FORK 

* In xv6 the **system call fork** simply calls the function named as **fork**.

    ```c
        int sys_fork() {
            return fork()
        }
    ```

* The fork system call is defined in **proc.c file**. The thing that forks 
  does to gets the current process proc structure which it needs to copied.

    ```c
        int fork(void) {
          int i, pid;
          struct proc *np;
          struct proc *curproc = myproc();
    ```

* Next call is to **allocproc which initializes a new proc structure** and
  initializes all the entries of the proc. Note this initialization is not 
  with respect to the process which needs called forked.

    ```c
        // Allocate process.
        np = allocproc()
    ```

* Now for the child process **all set of pages in the parent process needs to
  copied by allocating new pages** for child process. **copyuvm** function does
  the work of copying all the page tables.

    ```c
        np->pgdir = copyuvm(curproc->pgdir, curproc->sz)
    ```

    + copyvum first calls setupkvm function to allocates kernel pages and create 
      pages directory table with corresponding kernel page table entries.

        ```c
            // Given a parent process's page table, create a copy
            // of it for a child.
            pde_t* copyuvm(pde_t *pgdir, uint sz) {
                pde_t *d;
                d = setupkvm()              /* allocate the kernel pages */
        ```

    + Next job of copyuvm is **simiply copy parent data/text to new pages.** 
      For copyig create new pagedir for child and copy the contents by setting 
      up appropirate pagetables entries. **Copy whole process** thus size of the
      parent process is passed parameter.
                    
        ```c 
                pte_t *pte;
                uint pa, i, flags;
                char *mem;
                
                for(i = 0; i < sz; i += PGSIZE) {

                    pte = walkpgdir(pgdir, (void *)i, 0)    /* get a page table entry of parent process */
                    pa = PTE_ADDR(*pte);                    /* the page table entry address */
                    flags = PTE_FLAGS(*pte);                /* flags of the page table entry */

                    mem = kalloc()                          /* memory for child process */
                    memmove(mem, (char*)P2V(pa), PGSIZE);   /* copy the contents from physical address */

                    mappages(d, (void*)i, PGSIZE, V2P(mem), flags);     /* map the pages in the child pgdir */
                }
                return d;
            }
        ```

* **Copy the size** of the parent process to be same for the child process.   

    ```c
        np->sz = curproc->sz;
    ```

* **Modify the parent pointer** of child process to pointer to the parent process

    ```c
        np->parent = curproc;
    ```

* **Copy the entrie traframe** of parent process. However we need to **modify 
  the trap frame return value** which is zero for the child process.
    
    ```c
        *np->tf = *curproc->tf;
        np->tf->eax = 0;            /* clear the eax register -> return value for child */
    ```
    
    + **Question by whom this trapframe was set and where doesn't it return ?**
        
        + The initialization of the trapframe happens in allocproc, where the
          return value is specified to be forkret.

        + However just as discussed above the allocated traframe for child is
          now copied with values from parent process, which returns to the
          where the fork() call was made.
        
        + Note return value for child is modified in the traframe accordingly.


* **Duplicate all the file table values** from parent process.
  **filedup** system call is used here.

    ```c
        for(i = 0; i < NOFILE; i++) {
            if(curproc->ofile[i]) {
                /* dup for ith file structure */
                np->ofile[i] = filedup(curproc->ofile[i]);      
            }
        }
    ```

* **Duplicate the current working directory** of the parent process.

    ```c
        np->cwd = idup(curproc->cwd);
    ```

* **Copy the name of the parent process**

    ```c
        safestrcpy(np->name, curproc->name, sizeof(curproc->name));
    ```

* **Mark the process state of child to be RUNNABLE**

    ```c
        np->state = RUNNABLE;
    ```
     
* **Return the pid of the child process**

    ```c
        pid = np->pid;
        return pid;
    ```

## EXEC 

* The system call exec is defined in **sysfile.c**. However notice the system 
  call to exec doesn't take any arguments. However is exec one must be pass 
  the arguments to exec specifying the executable file name.

* **Note : exec being system call is called to in context of kernels memory, 
    Now the kernel needs to figure which file name the actaul exec was called**
    
    + Normally kernel nevers reads any user process memory however, but for 
      exec system call the kernel code needs to read the user memory. 

    + However keep in mind the before system call exec the arguments would have 
      been pushed on to the user process stack.  


* First thing which is done to **fetch the arguments to exec** and store the 
  values in some memory of kernel. Note you are in kernel context and reading the 
  user process memory. This work of fectiching the entries is done by **argstr**
  and **argint** functions.

    ```c
        int sys_exec(void) {
          uint uargv;
          char *path;

          argstr(0, &path);
          argint(1, (int*)&uargv);
    ```

    + **Both the function calls extract the data from the user stack**.

    + The function **argint** is defined in **sysfile.c** and fectchs the n'th 
      integer from the user stack, pointed by esp. Note extra 4 bytes are added 
      to skip the eip saved before the call. Addition is done because values on 
      stack are stored in reverse order.
    
        ```c
            // Fetch the nth 32-bit system call argument.
            int argint(int n, int *ip) {
                return fetchint((myproc()->tf->esp) + 4 + 4*n, ip);
            }
        ```
    
    + The function **argstr** is defined in **sysfile.c** fetches the string
      address passed as arguments. 

        ```c
            int argstr(int n, char **pp) {
                int addr;
                argint(n, &addr);
                return fetchstr(addr, pp);
            }
        ```
* Now next sys exec repeateadly copies all the strings  from the user stack to
  the kernel memory. The finally there is return to exec function. Basically 
  **work of system call exec was to create the arguments passed to user function 
  on the kernel memory and then call exec with the appropriate arguments**.

    ```c
            char *path, *argv[MAXARG];
            memset(argv, 0, sizeof(argv));
            for(i=0;; i++) {
                if(i >= NELEM(argv))
                    return -1;
                if(fetchint(uargv+4*i, (int*)&uarg) < 0)
                    return -1;
                if(uarg == 0){
                    argv[i] = 0;
                    break;
                }
                if(fetchstr(uarg, &argv[i]) < 0)
                    return -1;
            }
            return exec(path, argv);
        }
    ```

### What should exec do ?

* Remember, exec() comes after fork(), thus struct proc exists implying, page
  directory entries, kstack, trapframe, pid, etc.

* The exec code needs to read the ELF file mentioned in argv[0].

* **Create a new page dir, but discrad the old one**, and update the entries
  appropriately for user code and data. Copy the data from ELF file to the 
  pages which are allocated.

* Copy the arguments passed to passed exec in kernel memory's to user stack 
  allocated in one of the pages.


## EXEC (loader for xv6)

* The exec code in written in **exec.c** file which seperates it form the
  actual kernel code, since exec is loader program. 
    
* The first thing that exec does it **gets the inode for the ELF file** which needs
  to be executed. **inode is structure which defines the file attributes and
  data associated with the file**.

    ```c
        int exec(char *path, char **argv) {
            struct inode *ip;
            ip = namei(path);       /* basically get the inode of the file */
    ```

* Now the kernel is going to **read the ELF file**, why do this -> to figure out 
  size of the file, the program headers, etc. Also along with this xv6 checks
  if it is exactly an elf file or not.

    ```c
        struct elfhdr elf;
        readi(ip, (char*)&elf, 0, sizeof(elf));         /* read the ELF file headers */
        if(elf.magic != ELF_MAGIC) {                    /* check if it is actually ELF file */
            goto bad;
        }
    ```

* Next we allocate memory to **set the kernel pages** for the new process.

    ```c 
        pde_t *pgdir;
        pgdir = setupkvm();
    ```

* **Reach the program headers locate and copy each code of ELF file** into 
  main memory by allocating pages and updating the page table entries. 

    ```c
        sz = 0;
        struct proghdr ph;

        /* Load program into memory */
        for(int i = 0, off = elf.phoff; i < elf.phnum; i++, off += sizeof(ph)) {

            /* read the ELF file table header and locate the actual address of code */
            readi(ip, (char*)&ph, off, sizeof(ph));

            /* allocate the required pages */
            sz = allocuvm(pgdir, sz, ph.vaddr + ph.memsz);

            /* load the data from the file to appropriate pages */
            loaduvm(pgdir, (char*)ph.vaddr, ip, ph.off, ph.filesz);
        }
    ```

    + **allocuvm** allocates the pages in **pgdir** for code/data to be copied.
       returns the new size which is updated from the oldsize. 

        ```c 
            /* Allocate page tables and physical memory to grow process from oldsz to
             * newsz, which need not be page aligned.  Returns new size or 0 on error.
             */
            int allocuvm(pde_t *pgdir, uint oldsz, uint newsz) {
                
                char *mem;
                uint a;
                a = PGROUNDUP(oldsz);

                /* loop for creating sufficient pages */
                for(; a < newsz; a += PGSIZE){
                    
                    /* allocate a page */
                    mem = kalloc(); 

                    /* initialize the page */
                    memset(mem, 0, PGSIZE);

                    /* mappages addes the entry in the page directory / table */
                    mappages(pgdir, (char*)a, PGSIZE, V2P(mem), PTE_W|PTE_U);
                }

                /* returns the new size */
                return newsz;
            }
        ```

    + **loaduvm** loads/copies the code into pages which are allocated.
        ```c
            /* Load a program segment into pgdir.  addr must be page-aligned
             * and the pages from addr to addr+sz must already be mapped.
             */
            int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz) {

                uint i, pa, n;
                pte_t *pte;
                
                /* loop to load all the code at appropriate pages */
                for(i = 0; i < sz; i += PGSIZE) {
                    pte = walkpgdir(pgdir, addr+i, 0);
                    pa = PTE_ADDR(*pte);
                    if(sz - i < PGSIZE)
                        n = sz - i;
                    else
                        n = PGSIZE;
                    readi(ip, P2V(pa), offset+i, n);
                }
                return 0;
            }
        ```

*  We would be **allocating a guard page before we allocating the page for
   stackframe** of user process. Thus we first allocate space for two more pages

    ```c       
        sz = PGROUNDUP(sz);
        sz = allocuvm(pgdir, sz, sz + 2*PGSIZE);
        clearpteu(pgdir, (char*)(sz - 2*PGSIZE));
    ```

    + The clearpteu function basically clears the PTU\_E entry making the gaurd
      page inaccessible read and write.

        ```c
            void clearpteu(pde_t *pgdir, char *uva) {
                pte_t *pte = walkpgdir(pgdir, uva, 0);
                *pte &= ~PTE_U;
            }
        ```

* Now push the setup the stack frame of the process on the stack. The 3 is
  added in the stack to skip the return eip, argv and argc values.

    ```c
        uint ustack[3+MAXARG+1];
       
        // Push argument strings, prepare rest of stack in ustack.
        for(argc = 0; argv[argc]; argc++) {
            /* decrement sp */
            sp = (sp - (strlen(argv[argc]) + 1)) & ~3;
            /* copy the string from argv to sp */
            copyout(pgdir, sp, argv[argc], strlen(argv[argc]) + 1);
            /* step */
            ustack[3+argc] = sp;
        }
        ustack[3+argc] = 0;
    ```

* Initialize the return eip, argv and argc values 

    ```c        
        ustack[0] = 0xffffffff;  // fake return PC
        ustack[1] = argc;
        ustack[2] = sp - (argc+1)*4;  // argv pointer
    ```

* Copyout the above three values on the user stack.

    ```c
        sp -= (3+argc+1) * 4;
        /* copyout retrun ip, address, and number of 
         * arguments from ustack to sp 
         */
        copyout(pgdir, sp, ustack, (3+argc+1)*4);
    ```

* Now finally **save the old pagedirectory pointer**, and **set the process 
  page directory pointer to new pagedirectory allocated** 

    ```c 
        oldpgdir = curproc->pgdir;
        curproc->pgdir = pgdir;
    ```
        
* The current size of the process is updated, along with traframe where the
  eip and esp are modified accordingly.
    ```c
        curproc->sz = sz;
        curproc->tf->eip = elf.entry;  // main
        curproc->tf->esp = sp;
    ```
       
* **switchuvm** switched the TSS and h/w page tabels to corresponds to current process

    ```c
        switchuvm(curproc);
    ```

* Final call is to free the earlier allocated pages to the process. **freevm** 
  frees every allocate entry in the page directory table.

    ```c
        freevm(oldpgdir);
    ```
    
    + The code for freevm basically iterates over the page directory and free
      very page table which is allocated.

      ```c
            void freevm(pde_t *pgdir) {
                uint i;
                
                /* deallocuvm deallocates all the user pages */
                deallocuvm(pgdir, KERNBASE, 0);
                
                /* now free all the page tables entries present in directory */
                for(i = 0; i < NPDENTRIES; i++) {
                    if(pgdir[i] & PTE_P) {
                        char * v = P2V(PTE_ADDR(pgdir[i]));
                        kfree(v);
                    }
                }

                /* finally free the page directory */
                kfree((char*)pgdir);
            }
      ```

    + The **deallocuvm** deallocates the user pages which are allocated for 
      virtual memory address 0 to KERNBASE. Note the user pages are check if 
      part of the process before deallocation.

      ```c
        int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz) {
            pte_t *pte;
            uint a, pa;
            a = PGROUNDUP(newsz);
            for(; a  < oldsz; a += PGSIZE) {
                
                /* walk the page directory check if there is entry for the page */
                pte = walkpgdir(pgdir, (char*)a, 0);
                if(!pte) {
                    a = PGADDR(PDX(a) + 1, 0, 0) - PGSIZE;
                } 

                /* if there is entry and the page is allocated for process then deallocate */
                else if((*pte & PTE_P) != 0) {
                    pa = PTE_ADDR(*pte);
                    char *v = P2V(pa);
                    kfree(v);
                    *pte = 0;
                }
            }
            return newsz;
        }
    ```

