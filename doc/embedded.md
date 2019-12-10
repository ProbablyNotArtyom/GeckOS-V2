#  OS/A65 Embedded Application Notes

##  (c) 1989-98 Andre Fachat

* * *

### OS/A65 Embedded Version 2.0

#### Introduction

Embedded applications normally don't have the amount of RAM and ROM that a
"normal" (whatever this means) system has. Therefore I have tried to make some
parts of the kernel removable, allowing for a much smaller kernel to fit into
embedded applications.

This way the more interesting features are still available, like
multithreading, signals and the device code. Some of the more sophisticated
kernel features can be removed however. For how to do this see the
[config](kernel.html#config) section of the kernel description. The
application can be tested on a full-featured system while the embedded kernel
provides compatibility where necessary.

Parts of the kernel that can be removed are

  * streams 
  * memory management 
  * file manager 
  * send/receive 
  * semaphores 

This shortens the kernel to almost 2k in size. Also the used amount of RAM is
$bc = 188 byte normal RAM, plus $e = 14 byte zeropage (not counting the
environment, task and thread-save 6 bytes from addresses 2-8), also not
counting the possible use of PCBUF by fork and devcmd. 1k RAM total should now
be sufficient for simple applications.

Using the PCBUF without locking with a semaphore should be handled with care,
however.

