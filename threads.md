# THREADS

* **Thread - basic unit of CPU untilization**. A thread contains the three things
    + Program counter
    + General purpose Registers
    + Stack

* Any process can contains **mutiple threads which execute concurrently**. The
  threads shares the three things 
    + code section (text section -> program)
    + data section (globals/static variables)
    + heap section (dynamically allocated variables)

* Any traditional **heavy-weight** process has single thread of control. Process
  with multiple threads can perform more than one task concurrently. Thus
  thread is basically **light-weight** process 

    + The main reason why thread is light-weight process is that creating
      actual new process is time consuming and resource intensive as compared
      with creating new thread.

* **Web servers** are all multithreaded, which helps in giving responses to
  the millions client requests concurrently.

* Benifits of multithreading are 
    + Responsivenss 
    + Resource sharing 
    + Economy 
    + Scalability



# Threading Issues 

### fork(), exec() system call 

* **Problem** - In multithreaded program if there is system call to fork(), then 
  should the child process duplicate all the threads of the parent process ? or 
  just the thread which invoked the call to the fork() function ?. 

* **Question** - What to do in case of forking address space having multiple 
  threads executing at the time of the fork() system call.  

    + **In multithreading the threads are not necessarily aware of each other 
      thread's presence, purpose, actions, and so on**.

    + Example there are one thread called **deduct\_money()** which is running 
      along with **server** thread **concurrently**. Now the main thread calls the 
      fork() system call, If fork() during the new process creation, in multithreaded 
      enviornment would have copied all the active threads, then the **deduct\_money**
      thread would have been also copied and executed by the child process to be created.
      Thus the **deduct\_money** will be called twice.  

        ```c
            /* deducts given amount of money from account with given ID     */
            void deduct_money(int acount_id, int amount) {
                ....
                ....
            } 
            
            /* the main server thread which listens for requests            */
            int server(void) {
               
                while(true) {

                    /* wait for the message to be recieved                  */
                    message = recieve_message(); 

                    /* create thread to allow routine execute concurrently  */
                    thread_create(deduct_money, message.account_id, message.amout);
                    
                    /* duplicating the same process                         */
                    fork();         
                }

            } 
            
            The implementation of fork() system call would determines fate of
            the execution of the program. If fork() system call duplicates all
            the threads then the amount will be deducted twice from account.

        ```

    + In general these problems are of **threads modifying persistent state**. 
      **SOLUTION** : POSIX defined the behavior of fork() in the presence of 
      threads to duplicate only the forking / system calling thread. This will
      avoid improper changes being made to persistent state.


* **Question** - same question arises in implementation of exec system call,
    should the thread calling exec be replaced with the new process space ?  
    or alternatively should all the threads be replaced by new process space ? 
    
    + **SOLUTION** - the **exec()** system call works typically in the same
      manner basically will replace the **full process space** with new process
      space as mentioned in exec. Thus **all the threads in the currently
      executing in process would be replaced** by the new address space.







