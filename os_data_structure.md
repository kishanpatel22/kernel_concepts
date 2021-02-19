## OS data structures 

## PROCESS 

* **Process** is an **instance of the running program !**
* **Process** is **sequence of bytes occupying same space inside RAM** and is 
  **under the execution of CPU or is waiting for its execution on CPU**.
    + **Note : PROGRAM != PROCESS** A program is passive entity, such as list
      of instruction stored in a file (executable like ELF files) on disk.
    + **Question : When does the program become process ?** We need a **loader**,
      which can consult OS to load the program into the RAM and let OS schedule
      it's execution on CPU.

* Process has a definite lifetime.
* In UNIX all the process in form of well defined hierarchy. Each process has
  atleast one or more children but always has one parent. This the results in
  a tree like **data structure** with **init** process at the root of the tree.
  **init** is the first user process which is created when the system boots

## PROCESS STATES 

* In almost all conditions UNIX process are in well defined states
* Transition between any two process can be imagined as **finte state transition machine**

| **Process States**    | **Significance** |
|-----------------------|------------------|
| **New state**         | Process created in the secondary memory, still not picked by the OS. |
| **Running state**     | A process which is currently running on the CPU |
| **Waiting state**     | In such blocking situation(example as I/O) process call sleep() which changes process state to waiting |
| **Ready state**       | Process loaded in memory but is waiting for OS to schedule CPU time | 
| **Termination state** | process finished its execution |

* Each process has well defined context containing all the information.
    + User address space 
        + the program text, data, user stack, shared memory regions and so on
    + Control information
        + kernel uses two main data structures to maintain control information
            + u area
            + kernel stack 
    + Credentials / Accounting information
        + The credentials of the process include the user and group IDs
          assosicated with it.
    + Environment 
        + These are set of strings.

* **User address space**
    + Every user in the system is identified by a unique number called the user
      ID, or UID. Also there is a unique user group ID called as GID.
    + The root user or super user has the process UID 0 and GID 1.

* **uarea and proc Structure**
    + The proc structure - A process is represented with a **data structure 
      called PCB** (process control block).
    + One PCB per process and the OS maintains a "list" of such different PCB's
    + The underlying implementation of the list is of the form of arrary
      which contains the pointer to the process table and is generally of fixed
      size which means that at time instant there is a limit on the number of
      process that exists
    + Note that proc struture is in the system space and it is visible to the
      kernel everytime, even when process is not running.
    + The u area is always mapped with some fixed virtual address, and is
      visible to the kernel only when the process is runing.


## PROCESS ENTITY DATA STRUCTURE IN XV6

* **Structure of process is represented as PCB**
* Given below is the elements present in structure of xv6 kernel PCB, present in **proc.h** file.

```c

/* process states */
enum procstate { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

/* structure of the process */
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  int pid;                     // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};

```

* Each piece of the PCB can be classified as given below

|**Part of PCB**|**Significance**|**proc struct variable**|
|-----------------|------------------|--------------------------|
| Process state | Current state of the process  |  enum procstate state   |
| CPU register | General purpose registers need to be saved in context switch | struct trapframe \*tf, struct context \*contentxt | 
| CPU scheduing info | Information for the scheduling process (describes priority) | |
| Memory Management info | Pages tables, memory size, etc used by process | unit sz, pde\_t \* pddir |
| Accounting info | Amount of CPU time, real time, process number, etc | int pid, char name[16] | 
| I/O info | List of open files, I/O devides, etc | struct file \*ofile[NOFILE]  |


##### PROCESS CONTROL BLOCK REPRESENTATION INSIDE LINUX KERNEL

* The process control block in the Linux operating system is represented by
  C data structure **task_struct** which is found in the linux/sched.h file

```c
long state;                     /* state of the process */
struct sched entity se;         /* scheduling information */
struct task struct *parent;     /* this process’s parent */
struct list head children;      /* this process’s children */
struct files struct *files;     /* list of open files */
struct mm struct *mm;           /* address space of this process */
```

* Inside the kernel all the active process are represented as active **doubly 
  link lists** of the type **task_struct**
  
* Apart from the head and tail of the double link list the kernel maintains a
  **current pointer**, which points the process which is currently being executed
  by the CPU and the kernel and manipulate the state of the current process.
```c
    current->state = new_state;
```

## Process Scheduling 

* The main objective of multiprogramming is to have **some process running all
  the time** to increase the CPU utilization.
* There will be more process waiting to run at single time, and for this there
  has to be **ready/scheduling queue**.

* **SCHEDULING QUEUES** 
    + The queue contains the list of all the ready process, waiting for CPU execution.
    + The implementation of ready queue can be **link list type data structure**, with 
      head and tail pointers pointing to the first and last PCB blocks. The PCB
      itself contains self referential pointers pointing to the next PCB.
    + The process which is currently being executed may be interrupted (example
      - timer interrupt) or the process is waiting for some I/O (example - disk
      I/O). In such cases the OS has to change the state of such process to
      ready and make them wait in the scheduling queue.
    + **Each device has its own device queue**. The common representation is done
      by the **queuing diagram** 
      ![Queuing Diagram](https://slideplayer.com/slide/13723507/85/images/8/Representation+of+Process+Scheduling.jpg)
    + Each new process is initially is in the **ready queue**. It wait there
      untill it is allocated CPU time, after which there can occur
        + **I/O request** which cause the process to wait in the **I/O device queue**
        + Process **creates child process** and can **wait for child to terminate**
        + Process is forcibly removed from the CPU, as result of an interrupt
          and put back in ready queue.

#### SCHEDULAR 




## CONTEXT 

* When an **interrupt occurs**, the kernel needs to save the current context of
  the process which is running on the CPU. 
  
* **Why save context ?** -> the reason is process can restore its previous 
  context when it gets CPU resource allocated from the kernel. 

* **Question what is the context of process ?** -> its the **PCB of the process**

* The kernel needs to switch the CPU, to another process and for this, it needs
  to **save context of current process and restore context of another process**. 
  This is known as **context switch**.

* The **context switch time is a pure overhead**, since no useful work is done
  during this time.



