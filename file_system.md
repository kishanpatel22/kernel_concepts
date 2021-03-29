# File Systems

* File system is part of the kernel which manages the disk hardware and implements
  user level view of files and folders which are stored on the physical disk.

* The implementation of file system can vary, depending upon what type of data
  structures the kernel is providing for managing files, why type of convention
  is used for physically storing data on disk, etc


## Partitions 

* It is logical division of the volume grounp which consists of set of physical 
  volumes. The logical division made by kernel helps in effectively creating multiple
  regions from the given set of physical disks.

## Formatting 

* The process of creating the initialized data structure on paritition, so that
  one can start creating the files and folders on the paritions based upon the 
  data structure used by kernel like **acyclic graph data structure**.


## File system on Linux

* The /proc/partitions file on the system is psuedo file system reflection
  of the kernels implementation of file system.

* There has to be atleast one partition on the harddisk, thus any harddisk must
  essentially have parition table. The one partition may span whole hard disk.
  However all one can may logical division inside the parition as well.






