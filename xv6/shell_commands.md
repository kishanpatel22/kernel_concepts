## xv6 OS task

#### Installation 
```sh
    $ git clone https://github.com/mit-pdos/xv6-public.git
    $ cd xv6-public
```
#### Build
```sh
    $ sudo apt install qemu 
    $ make qemu
    $ make clean
```
* Note make qemu complies and builds executable and helps to enter the qemu 
  console terminal and make clean will help you remove the .o files
  generated during compliation process

#### Basic command and application codes

| **Command** | **System Calls**   |
|-------------|--------------------|
| cat         | read() call with file descripter and write() for console  |
| ls          | open() call and using fstat() to read file status based upon the file system |     
| echo        | printf() call which inturns calls write() after parsing arguments |         
| rm          | unlink() call to remove the file  |             
| ln          | link() call to create new file hard link |         
| grep        | read() call with file descripter and write() for console after pattern matching |         
| kill        | this will need locks usings calls of acqurie() and release() before killing process |         
| wc          | open() call with file descripter and write() for console after counting characters, words and lines |         
| mkfs        |                                                          |         
| mkdir       |                                                          |         

#### Task - Running 20 different commads and checking the output 

* The given below command will print the file contents on the console
```sh
    $ cat REAMDE
```
Some interesting things to do here is that run **$ cat cat** this will print
all the binary content of the cat executable file. Note that its very simple
cat function written to display the concatenated file and not advanced cat as
written for linux.

* echo command is very useful in displaying as text on the console.
```sh
    $ echo foo bar baz
```
This display foo bar and baz on the console of qemu


* Now since there is no touch command you need to create file using > (redirection)
```sh
    $ echo foo bar baz > newfile.txt
    $ cat newfile.txt
```

* Some **noticable changes** using the redirection operator. In the previous
  command foo bar and baz are written in the new file. Now if you run command
```sh
    $ echo new foo > newfile.txt
    $ cat newfile.txt
```
This command will overwrite the pervious content already written in the new
file, which means unlike linux command where it truncates the file contents and
then writes the content the implementation of the redirection in xv6 is simply 
start from filepoition at **SEEK SET** which is the starting position of the
file and write the overwrite the contents from there.


* No appending redirection present in xv6
```sh
    $ echo new bar >> newfile.txt
    $ cat newfile.txt
```
The above command works the same as redirection unlike linux command which
append the new contents are the **SEEK END** which is the end of the file.


* remove file from the console
```sh 
    $ rm newfile.txt
```
One interesting command to write on xv6 is **rm rm**, note do this at your own
risk. However this reminds me of thanos saying **I used the stones to destory
the stones**. One more point about rm there is no regex as in linux, it very
simple code written hardly 50 lines do check that out. Note the underlying
implementation of rm calls the **unlink** system call.


* Create hardlinks on xv6 using ln
```sh
    $ ln newfile.txt file.txt
```
This commands creates hardlink with a new filename newfile.txt which has the
same inode value as the file.txt (note that there is no concept of original file
in the hardlinks, all of hardlinks actually point to the same and have same
inode, just they different names)

* The famous grep command displays the line where the word is found
```sh
    $ grep run README
    $ grep ^foo file.txt
    $ grep baz$ file.txt
    $ grep r* newfile.txt
```
Note the underlying grep implementation expects the first argument as patter
and subsequent arguments to it can the file names in which the pattern needs to
be found. The grep function is taken from Denis Riche book where it shows a
simple evalutaion of **grep pattern matching using recursive function**.


* Now we can pipe the commands output to the other commands as input as
  normally done in gnome terminal
```sh
    $ cat REAMDE | grep run | wc 
```
This display the number of lines where the words run ocurres along with word
and charavter count


* Kill process with given process ID
```sh
    $ kill 100
```
The above command will kill the process with id equal to 100, however note it
will be **interesting to kill process with id equal to 1**, just type this can
check the kernel code will panic and lead to an infinte loop the simiple reason
is that when bootloading process is over and control is handed to kernel, then
its the kernel who start's the process with id equal 1 (the first process) of
init process and from which the other process are derived. Killing the process
with id equal to 1 implies killing the kernel process which is responsible for
starting and shutting down the system.

#### Task - Questions on xv6

* **How is a user program compiled ?**
    + Normally user programs get complied directly into the machine level executable 
      code using some good compliers like **gcc**. But what happens is that the
      underlying complier executable code knows the machine platfrom (OS + hardware)
      by using OS kernel systems calls. 
    + However in case of platforms with xv6 where there is no complier
      executable code present for the platform, thus the user programs need
      something called as **cross complilation**.
    + In such senarios what you need to do is use existing **gcc** or some good
      complier to complie your code for some other platform like xv6 in our case.

* **How is the kernel itself compiled ?**




