# xv6 scheduler

* **Schedular** code is written as ISR by kernel for **timer interrupts**, and
  sometimes may even get invoked in case of **Page Faults**.

* **Schedular work is schedule the next process to execute on CPU**. Bascially 
  **scheddular selects a paritcular process in ready state and changes the
  state from ready to running by giving the control of CPU to process**

* Schedular is special part of the kernel code, which has to be written in
  assembly langauge, since it has to **breaks the calling convention**.

* Note that scheduler is **independent piece of code** in kernel, it requires
  **different stack in kernel for its execution**. 

  |          **STACKS**           |    Example    |
  |-------------------------------|---------------|
  | process context process stack |  any process  |
  |  process context kernel stack |  interrupts   |
  |  system context kernel stack  |  scheduler    |

* **Question - why does scheduler requires different stack part from kernel stack used for process ?**
    + 


* Now there is going to be change of 4 stacks when timer interrupts occur.


* select a process to execute 





