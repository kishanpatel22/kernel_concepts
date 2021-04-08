# MIDDLE LAYER FILE SYSTEM HANDLING CODE

## LOGGING CODE

* Logging is mechanism of recovery from the physical disk.
    + recovery when there are inconsistent data structures of file system due 
      to power failure, umounting, dirty writes not written on disk.
    + multiple operations needed for system call (like open) for which some
      writes succeed and some don't

* In the log all the changes for a **transaction** (an operation) are either
  written completely or not at all.

* For recovery, completed / commited operations must be rerun, and incompleted 
  operations must be neglected.

* xv6 system call does not directly write the on-disk file system data
  structures, Rather a system call named begin\_op() and end\_op() are made 
  for starting and starting and commiting transaction.

    + begin\_op increaments log.outstanding.
    + end\_op decreaments log.outstanding and if it is 0, then calls, commit()

* During the code of system call if the cache buffer is modified, and done with 
    + log\_write() is called
    + This copies the block in an array of blocks inside log, the block is not
      written in it's actaul place in FS as of now.

* When finally commit is called, the modified block is written back to the disk.



