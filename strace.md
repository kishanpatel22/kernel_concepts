# Strace LINUX command

* Strace linux commands helps to trace the system calls made by the programs.
  Strace uses Ptrace functionality of linux kernel wherethe "tracer" is able to 
  trace the systems calls made by the tracee (based upon signals gernerated).

* Some examples of using strace command are given below
  
```sh
    $ strace ls             # this will trace all system calls done by ls command 
    $ strace ./my_exe       # this will trace all system calls done by my exe ELF file
    $ strace -c ls | wc     # display the summary for the sequence of commands
```

* One can try even trace strace command by running strace over strace which
  is very cool.
```sh
    $ strace strace ls      # this will trace the strace
```

## TRACK of system calls



