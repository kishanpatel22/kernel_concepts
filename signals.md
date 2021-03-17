# SIGNALS

* **Signal is a software interrupt delivered to a process by another process**.
  Kernel uses signals to report exceptional situations to an executing program.

* Some signal report errors like invalid memory, buffer overflow, etc, while
  some signals report asynchronous events, which may be harmless.

* There are variety of signals which are supported by the GNU C library and if
  user wants to take paritcular action for some specific signals then he needs
  to define **signal handler**. The kernel in it's data structure keeps the
  records of all such signal handlers, in case of user defined signal handler
  it overwrites the default signal handlers.

## Kinds of Signals 

* **Signal reports the occurrence of an exceptional event**. The events in
  process can raise / generate signals accordingly.

    + divide by zero condition                  
    + termination of program \<control-C\>          
    + expiration of timer or alaram             
    + kill another process
    + I/O operations signal 

* Note that signals are limited but can perfrom **Inter process communication**(IPC)



