# Compliation Flags in GCC

* **Compliation Flags** or **CC FLAGS** are given to complier to perform
  certain type of activity during the process of compliation. There are variety
  of flags in the gcc linux manual, some of flags used in xv6 compliation are 
  given below

* **-ggdb** 
    + -g flag helps in creating additional information in the EFL file which
      can be used by GDB (GNU debugger) for debugging our program. However 
      -g only produces debugging information in the operating system’s native 
      format (stabs, COFF, XCOFF, or DWARF)
    + -ggdb Produce debugging information for use by GDB.  This means to use 
       the most expressive format available (DWARF, stabs, or the native format
       if neither of those are supported), including GDB extensions if at all
       possible.
    + GDB is very useful tool to debug the code, as it provides functionality to
      line by line or even instruction by instruction track your program 
      variables and even registers. It provides many usesfull functionalities 
      that are mentioned [here](http://web.mit.edu/gnu/doc/html/gdb_toc.html)



