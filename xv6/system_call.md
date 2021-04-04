## SYSTEM CALLS




### ADDING SYSTEM CALLS to xv6

* For adding any system call to xv6 one needs two things, create a user land
  test code for system call and make changes in kernel to accomodate the
  required system call

* **user.h** file contains prototypes of the wrappers of all the system call.
  The prototype of your system call needs to be mentioned in this file. Example 
   
    ```c
        int hello(void);            /* hello world system call */
    ```

* **usys.S** contains an assemble level code and #define macro for making
  system call. The define inokes for any system call with the given name using
  the int instruction. The value $SYS\_##name is again macro specifying the
  index in the array of system call.
        
    ```asm
        #define SYSCALL(name)               \
        .globl name;                        \
        name:                               \
            movl $SYS_ ## name, %eax;       \
            int $T_SYSCALL;                 \
            ret
    ```

    ```asm
        SYSCALL(hello);         /* this defines the system call */
    ```

* **syscall.h** contains marcos for the index in array of system call function pointers.

    ```c
        #define SYS_hello  (22)
    ```

* **syscall.c** contains the array of system call function pointers.
    
    ```c
        extern int sys_hello(void);             /* declaration of the system call */

        static int (*syscalls[])(void) = {
            [SYS_hello]     hello,              /* initialization of the function pointer */
        }
    ```

* **Questions : array contains function pointers to function returning int 
    and accepting void!! not all system calls implementation are like this**

    + When parameters are passed to the system calls they are pushed on to the 
      **user process stack**, however but on system call, the kernel runs on
      behalf of the user process, using the **kernel stack**.
    
    + So what happens here is there needs to be intermediatory function that
      would copy the data or the parameters from user process memory and push
      them to the kernel stack memory.

    + Finally the kernel runs the actual system call as if it was passed with 
      the user parameters.

    + **Note** not for all system calls seperate function needs to be created,
      example fork, since fork system call never takes any arguments.

* **sysfile.c** actaully contains the implementation of the system call.

