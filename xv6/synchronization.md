# SYNCHRONIZATION in xv6

## SPINLOCKS in xv6

* **Spinlocks** are the most primitive level locks that can be provided using 
  hardware feature of **atomic execution** of instructions like **test-and-set**

    ```c
        // Mutual exclusion lock.
        struct spinlock {
            uint locked;       // Is the lock held?
        
            // For debugging:
            char *name;        // Name of lock.
            struct cpu *cpu;   // The cpu holding the lock.
            uint pcs[10];      // The call stack (an array of program counters)
                               // that locked the lock.
        };
    ```

### initlock 

* Initialization of the spinlock structure, should be done once before 
  acquiring any locks.

    ```c
        void initlock(struct spinlock *lk, char *name) {
            lk->name = name;        
            lk->locked = 0;     // initially no process has lock on critical section.
            lk->cpu = 0;
        }
    ```
### acquire

* **Acquires spinlock for process on the critical section**.

    ```c
        void acquire(struct spinlock *lk);
    ```

* The first thing acquire does is to **disable interrupts** and saves the flag
  resgiter values.

    ```c
        pushcli(); // disable interrupts to avoid deadlock.
    ```

* If some **other process is holding the lock then the kernel will panic**.

    ```c    
        if(holding(lk))
            panic("acquire");
    ```

* The actual loop which does a **busy wait**, for the acquiring the lock.

    ```c
        // The xchg is atomic.
        while(xchg(&lk->locked, 1) != 0) {
            ;
        }
        // acquired the lock 
    ```

* Then there is memory barrier instructuion for the complier and then fillin 
  the rest parameters related to debuggin information

    ```c
        __sync_synchronize();
        
        // Record info about lock acquisition for debugging.
        lk->cpu = mycpu();
        getcallerpcs(&lk, lk->pcs);
    ```

### release 

* Release the acquired spinlock by the process.

    ```c
        void release(struct spinlock *lk);
    ```

* Reset the debugging infomation about the struct spinlock.

    ```c
        lk->pcs[0] = 0;
        lk->cpu = 0;
    ```

* Memory barrier instruction for the complier 

    ```c
        __sync_synchronize();
    ```

* Actaul atomic instruction for release the locked critcal section by moving
  zero to the locked varaible in spinlock.

    ```c
        asm volatile("movl $0, %0" : "+m" (lk->locked) : );
    ```

* Restore the interrupts.

    ```c
        popcli();
    ```







