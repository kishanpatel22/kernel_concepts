## UNIX OPERATING SYSTEM

* UNIX OS is referred as the "kernel" which is special type of the program that
  runs directly on the hardware and implements the process model and other
  system services

* UNIX provides functionality in these four ways
    + Providing system call API for software interrupts
    + Handling the asynchronous interrupts 
    + Unusal conditions in the program like divide by zero, overflowing, etc
      are handled by the kernel by it's intervention
    + A set of special system process such as swapper and pagedeamon perfrom 
      system wide task such as controlling number of active process in memory pool

* UNIX provides two modes of execution 
    + User mode   
        + only normal instructions available for executing on CPU
    + Kernel mode
        + normal + priveleged instrutions available for executing on CPU

* **Why need this seperation of modes ?**
    + Protection from accidentally or maliciously corrupting another process or
      the kernel. The damage done by any process can be localized and usually
      also have saves the user from getting unsual outcomes.

## Interrupts

* Interrupts basically change the normal sequence of execution inside CPU 
  (fetch decode and execute) and transfer the control of CPU (change of PC
  register) to service the interrupt.

* **Who** services the interrupt, its the kernel ! **How ?**  at the time after 
  bootstrapping is done the kernel call its init process which simply copies its
  own code to all possible locations where interrupts can occur. Basically
  manipulating the IVT and creating mapping for the appropriate ISR's

* The two types of interrupts are given below
    + **Hardware Interrupts**
        + Hardware devices raise the hardware interrupts. Such type of
          interrupts are asynchronous in nature i.e. can occur any time and any
          number of times.
        + Why need this ? Computer is made of many hardware devices which are 
          connected to the CPU motherboard and inorder for the devices to
          communicate they need to interrupt the normal CPU execution.
    + **Software Interrupts**
        + There are special type of instructions in assembly language which
          raise software interrupts ?
        + Why need this ? The user application code just executes in normal
          mode thus it doesn't get chance to execute priveleged instructions.
          Now user application code needs way to request kernel who with
          priveleged instrutions can do specify task.

* **Interrupts can also occur when the system is the kernel mode**. In this
  case the system will remain in the kernel mode even after handler completes.

## System Calls - Talking with the kernel 

* Any process cannot directly talk with the kernel, so there is any **API** 
  provided by the kernel in from of the system calls which help the user
  application code **to avail the services provided by the kernel**.

* System call become very significant becuase they are the ones which help to 
  switch from user mode to the kernel mode for doing a paritcular task. Note
  for this switch obviously the system call will raise a software interrupt.


## Kernel Execution Context 

* Now as system calls help in calling the kernel for providing a service, we
  say kernel execution happens on behalf of the **process context**. 
  Process telling kernel to do something 

* However some task are not performed by context of process, but the hardware
  devices rasing the hardware interrupts such situation the kernel is called
  on behalf of the **system context**







