#  OS/A65 Operation without MMU

##  (c) 1989-98 Andre Fachat

* * *

Using OS/A65 without an MMU used to require some special measures. One of it
was to coordinate all running tasks concerning their memory usage.  
With the new relocatabel o65 fileformat the lib6502 implemetation takes care
of that.

This means when building lib6502 programs, you do not have to actively care
about the stuff below, but it is probably good to know anyway.

There is one thing that's difficult to handle on a 6502, the stack. It's only
256 byte large (small that is...). And with a multitasking OS the stack has to
be used by several programs at the same time. One approach is to copy the
stack contents to another place every time a task switch is done. But that's a
big speed penalty. Instead I split the stack into 6 pieces, 5 of them for a
task and the last one for the system. So on a system without MMU, only five
tasks can be running concurrently.

With version 1.3.9 we got a new config option, STACKCOPY. With this option the
stack of a task is saved in some save area when doing a task switch. This is
slower, but allows much more stack space in a task and much more tasks.

There is one other thing that cannot easily be coordinated - the SEND buffer.
This buffer is used for communication with the kernel. As OS/A65 originated
from a system with MMU, there was no real need not to use an absolute address
for the buffer, thus saving registers for other purposes.  
For all systems now the usage of the SEND buffer has to be coordinated between
the tasks. Therefore a system semaphore is used.

    
    
    #define	SEM_SENDBUF	&lt-1;
    

Before accessing the SEND buffer at $02**, the task has to allocate it with a
PSEM operation on the SEM_SENDBUF semaphore. After the system call, it has to
be freed with a VSEM operation.

Well, and here we are with the known resource allocation problem. Imagine two
tasks communicating via send/receive. Then, if the order in which messages are
sent is not fixed and therefore predictable, precautions against a lock have
to be taken: task 1 allocating the send buffer to send a message to task 2.
The same time task 2 tries to allocate the send buffer to send a message to
task 1. If it not tries to receive messages while waiting for the semaphore,
it will lock.  
Therefore the filesystems, for example, do not release the send buffer while
executing a command or open a file. In the meantime another task could send a
message to the filesystem, locking the send buffer. But the buffer is needed
by the filesystem to send the reply message. The filesystems in their current
form are not prepared for this situation, so that they don't release the send
buffer.

Be careful when using the send buffer - you are warned!

The lib6502 library is thread-save. It can be used by any thread at any time.
Internal locks (semaphores) are used to avoid any interference if necessary.

* * *

Suggested reading: "Operating Systems, design and implementation", Andrew S.
Tanenbaum, Prentice-Hall

