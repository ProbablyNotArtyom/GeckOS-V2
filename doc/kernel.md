#  OS/A65 Kernel Interface Description

##  (c) 1989-98 Andre Fachat

* * *

### OS/A65 Version 2.0

#### Introduction

The change from kernel 1.3 to kernel 2.0 is radical in some things, but
conservative in others. The complete environment handling has been rewritten
to make it easier to port to different platforms. Also threads have been
introduced. Therefore all the memory management and interprocess communication
calls have changed as well. The scheduler is now a lot faster, as no more
checks are done for threads in the waiting list.

Although most routines have been rewritten, many of these calls still have the
same parameters and behave the same way. Also the `PCBUF` is still used
(unfortunately). This is the general communications buffer needed for
filesystem operation and some kernel calls. It is a global buffer, and as such
it is protected by the `SEM_SENDBUF` system semaphore. Each task that wants to
use has to allocate this semaphore with `PSEM` before using the buffer.

Also there still is no block oriented communication, although the stream based
communication has been improved by the out-of-band error, brk and push/pull
flags.

I try to keep this documentation as correct and up-to-date as possible, but if
in doubt, read the source... `:-)`

There are now some comments like "this need not be available in all OS
implementations". These comments mostly refer to the embedded versions of the
OS where the kernel can be shortened to (almost) 2k in size by leaving out
some stuff not necessary for the particular application.

#### Contents

  * Kernal Interface Description
  * Error Codes
  * ROM bootup
  * Variable Allocation
  * Configuration during build
  * Porting Information
  * System Constants

#### Kernel Interface Description

Most kernel routines return an error code in the accumlator (a), if the carry
flag is reset upon return. Otherwise the ac might contain data.

Please note that, while the addresses here are given as starting with $f000,
in the real system the address could be basically anywhere depending on the
architecture. The loader relocates them on the basis of the "OSA2KERNEL"
label: the file contains it as undefined label, and the loader relocates the
kernel addresses that are stored as offset to this base as appropriate.

* * *



    $f000  	RESET   System reset



* * *



    $f003	ENMEM	Enable 4k memory block, no in xr, for memory management -
    		MMU version only
    $f006	SETBLK	set memory block ac at MMU entry yr - returns old entry
    		in ac. MMU version only



* * *

The Stream functions have not changed much. The only addition compared to
kernel 1.x is the error code byte, that allows passing of error conditions and
push/pull signals as "out-of-band" data.

In the [embedded](embedded.md) versions the stream functions are not
necessarily implemented.




    $f009	GETSTR	Get a free stream. increase read and write task counter,
    		returns stream number in x

    $f00c	FRESTR	decreases read and write task counter, thus freeing the
    		stream, if both result in zero. parameter: stream number in x

    $f00f	PUTC	puts a byte on the stream. parameter: stream number in x,
    		data byte in a. returns errorcode in a

    $f012	GETC	Get a byte from the stream. parameter: stream number in x,
    		returns data byte in a (with carry=0) or error code
    		(carry=1)

    $f015	UNGETC	Gives byte from stream back. parameter: stream number in x,
    		data byte in a, returns error code in a

    $f018	STRCMD	executes a stream command. parameter: stream number in x,
    		stream command in a, returns error code in a.

    		possible stream commands are:

    		SC_GET		0	same as GETSTR
    		SC_REG_RD	1	increase read task counter by one
    		SC_REG_WR	2	increase write task counter by one
    		SC_CLR		3	empty the stream
    		SC_EOF		4	closing the stream from the sender
    					side, i.e. decrease write task
    					counter by one
    		SC_NUL		5	closing the stream from the receiver
    					side, i.e. decrease read task
    					counter.
    		SC_FRE		6	same as FRESTR
    		SC_STAT		7	read stream status into ac
    		SC_GANZ		8	returns number of bytes in stream in a
    		SC_RWANZ	9	returns the stream write pointer in
    					a and the stream read pointer in y

    		SC_ESTAT	10	Get the error code byte in yr,
    					and the normal (i.e. what you get
    					with SC_STAT) stream state in ac.
    		SC_SSTAT	11	Set bits in the error code byte
    					each bit set in yr is set in error byte
    		SC_CSTAT	12	Clear bits in the error code byte
    					each bit set in yr is cleared in error
    					byte


    		Possible stream status are:

    		E_NUL		Noone reading from stream
    				(i.e. read task counter is zero)
    		E_EOF		stream is empty and noone is writing anymore
    				(i.e. write task counter is zero)
    		E_SEMPTY	stream buffer is empty
    		E_SFULL		stream buffer is full
    		E_SLWM		number of bytes below Low Water Mark
    				(1/4 buffer size)
    		E_SHWM		number of bytes above Hogh Water Mark
    				(3/4 buffer size)

    		The error code byte has the following bits:

    		SCE_PULL	%10000000  reader pulls data from sender
    		SCE_PUSH	%01000000  writer pushes data to receiver
    		SCE_BRK		%00100000  writer sends 'break' signal
    		SCE_RERRM	%00001100  reader error condition mask
    		SCE_WERRM	%00000011  writer error condition mask



The error condition byte can be set and read from both ends. This byte is
cleared when the stream is allocated. If the reader "pulls" data, this is a
sign for the writer to flush all its internal buffers into the stream. When
this is done, the writer clears the pull flag.

If the writer wants to push his data (like after an CR on a terminal, for
example) the reader should get the number of bytes in the stream to have an
inidcation of how many bytes to read as "pushed" (or urgent in Internet
speak). Then it removes the push flag and gets the bytes from the stream

If an error occurs, e.g. a read error when reading a file, the writer sets all
the bits in the error code byte masked by SCE_WERRM to 1 (which is general
error code - other codes still have to be defined). The reader of the stream
then reads this and can handle the error.

If an error on the reading side occurs (like disk full when writing a file),
The reader sets all the error bits masked by SCE_RERRM to 1 (which is general
error code - other codes still have to be defined). The writer can then close
the file on his side.

If the writer wants to send a `Break` (i.e. a `^C` equivalent) code to the
reader, it sets the `SCE_BRK` bit. The reader must clear it itself.

* * *



    $f01b	DEVCMD	executes device commands. parameter: device number in x,
    		device command in a, optional parameter in y.
    		returns error code in a.

    		possible stream commands are:

    		DC_IRQ		0	execute IRQ routine
    					the interrupt routine has to return
    					E_OK if an interrupt source is cleared
    					or E_NOIRQ if not.
    		DC_RES		1	initialize device
    		DC_GS		2	sets the stream (in y) the device
    					reads from
    		DC_PS		3	sets the stream (in y) the device
    					writes to
    		DC_RX_ON	4	switch on receive
    		DC_TX_ON	5	switch off receive
    		DC_RX_OFF	6	switch on send
    		DC_TX_OFF	7	switch off send

    		DC_SPD		8	set speed (in y), device dependend
    		DC_HS		9	set handshake (in y), device dependend
    		DC_ST		10	get device status, device dependend
    		DC_EXIT		11	disable device, including IRQ sources

    		DC_GNAM		16	get the name of a device, parameter:
    					device number in x, returns
    					name PCBUF ($0200)
    		DC_GNUM		17	get number of a device from name,
    					parameter: name in PCBUF, length
    					of name in x, returns device number
    					in x
    		DC_REGDEV	18	register new device(s)



With the registration of the device the system has to know what to do with it
and where to put it in memory. Therefore you have to put the address of the
following struct into x/y registers.





    		x/y ->          2 byte pointer to label1
    				JMP Start_of_1st_device
    				Name_of_device,0
    				...
    		label1		2 byte pointer to label2
    				JMP Start_of_2nd_device
    				Name_of_2nd_device,0
    				...
    		labeln		$ffff



With this method it is possible to register several devices, as long as they
are in one memory block, with one call. The devices are started with DC_RESET.
In y they get a system feature byte. Currently only bit 7 is defined, where
$00 is 1 MHz system clock and $80 is 2 MHz system clock.

* * *



    $f01e	FORK	start a new task. parameter: yr = length of the following
    		struct in PCBUF

    		FORK_SIZE	0	The size of the needed memory
    					in 256 byte blocks, from botton up
    		FORK_SHARED	1	number of 256 byte blocks that
    					share the mapping with the current
    					task, from top down.
    		FORK_PRIORITY	2	priority of new task (0=inherit
    					from parent)
    		FORK_STDIN	3	stdin stream
    		FORK_STDOUT	4	stdout stream
    		FORK_STDERR	5	stderr stream
    		FORK_ADDR	6	start address of task
    		FORK_PAR	8	parameter (passed to new task in AC)
    		FORK_NAME	9	name of the program, ended by a
    					nullbyte. After that follows the
    					command line that started the program.

    		FORK returns the PID of the new task in xr
    		and the parameter from FORK_PAR in ac.


The commandline is put into the PCBUF of the starting task. The new task gets
its own task id in x The stream read/write task counter are not changed. I.e.
a thread that wants to use the end given to fork too has to increase the
read/write counters itself.

The priority is a hack to allow to run tasks with different execution
priorities. The priority counts the number of interruptions before a thread
switch occurs. Default value is 3. A value of 0 lets the new task inherit the
priority of the current task.

It is now somewhat difficult to really start a new task, as the READ/WRITE
routines to access another tasks memory have been removed.
One standard procedure could be to write a boot routine into the current
PCBUF, after the command line, and jump to this boot routine when the FORK
parameters are copied to the target PCBUF. The length parameter to FORK must
of course include this routine.
You could also use a routine in the ROM, which then should be shared between
the calling and the called task. The boot routine can then load the
executable.
For a stdlib with null-separated command line parameters, they may also be
included after the name, as long as the length byte includes them too.

In fact, the FORK_NAME entry is somewhat arbitrary at this time, as it is not
used within the kernel. In fact, one could write anything there. But to keep
compatible, the name and command line must have the same format as for the ROM
startup, i.e. a nullbyte separated list of program name and parameter, with a
zero-length parameter at the end.

The newly created task must release the SEM_SENDBUF semaphore for the PCBUF.




    $f021	TERM	<- a = return code
    		ends the current thread. If still threads in the tasks are
    		running, the return code is ignored. Otherwise the return
    		code is sent to the parent with SIGCHLD.

    $f024	KILL	<- a = return code, x = task id
    		ends a complete task, with all threads in them.
    		ac is return code, xr is task ID to kill.
    		The return code is sent to the parent with SIGCHLD.

    $f027	YIELD	just give control back to the scheduler, as there is nothing
    		to do at the moment - but keep running (not needed for
    		preemtive multitasking, an interrupt interrupts any task!)

    $f02a	FORKT	forks a new thread in the current task. Execution start
    		address must be in a/y.
    		returns c=0 for ok and new thread no in xr;
    		or c=1 for error (errno in AC).

    $f02d	SBRK	(carry=1): sets the freely available RAM area in the
    			   active environment to ac blocks with 256 byte
    			   each, by adding memory to or removing memory
    			   from the end of the already available memory
    			   area. Returns in ac the length
    			   of the available memory in 256 byte blocks,
    			   if c=0 on return. Otherwise returns error.
    		(carry=0): returns in ac the length of the available
    			   memory in 256 byte blocks.

    $f030	GETINFO	getinfo returns process information about all (up to 16)
    		running processes in the PCBUF (which must be allocated
    		by the process before)

    		May not be available in all OS implementations.

    		There are ANZ_ENV valid entries in the table, a
    		0 in the TN_NTHREADS field indicates an empty slot.

    		The fields are as follows:

    		TN_PID		0	PID of process in this entry
    		TN_NTHREADS	1	Number of threads for process
    		TN_ENV		2	Environment number for process
    		TN_PARENT	3	Parent process
    		TN_MEM		4	amount of allocated RAM
    		TN_SIGNAL	5	signal address
    		TN_STDIN	7	STDIN stream
    		TN_STDOUT	8	STDOUT stream
    		TN_STDERR	9	STDERR stream
    		TN_SIGMASK	10	signal mask value
    		TN_EXECADDR	11	task start address
    		TN_PRIO		13	task priority
    		TN_NAME		14	start of prog name (if available)

    		Each field is TN_SLEN bytes long and the
    		length of the name field is TNAMLEN. If the name is longer
    		than TNAMLEN-1, the the last characters are cut off, and in
    		the table the ending nullbyte is missing.

    $f066	SETINFO	setinfo updates the task table with the correct
    		start address of the active task (used by lib6502).

    		x   ->          field to update (must be TN_EXECADDR)
    		a/y ->          task start address


    $f033	DUP	(carry=1): set new STD* stream. parameter: STD* number in x,
    		new stream in a, returns old stream in ac;
    		read/write task counters are not touched!
    		(carry=0): get redirected STD* stream number. parameter:
    		STD* number in x, returns stream in a



* * *

The semaphore calls need not be available in all OS implementations.




    $f036	GETSEM	gets a free semaphore. returns semaphore number in x

    $f039	FRESEM	frees a semaphore. parameter: semaphore number in x
    		negative (system) semaphores cannot be freed (nor are they
    		returned by GETSEM)

    $f03c	PSEM	'PSEM' operation on a given semaphore. task waits till
    		semaphore is freed. parameter: semaphore number in x
    		carry=0: block till semaphore is free
    		carry=1: do a test&set; operation and return with
    		E_OK if semaphore gotten, or E_SEMSET if semaphore is in use.

    $f03f	VSEM	'VSEM' operation on semaphore, allows other tasks
    		to grab the semaphore.
    		parameter: semaphore number in x



* * *



    $f042	SEND	send a message to another task
    		parameter: optional message type in a, target task in x,
    		length of data in PCBUF in y; y=0 means 256 byte in PCBUF
    		The data is in PCBUF ($0200-$02ff). returns a and y as
                    given and the 'redirected' target (e.g. from filesystem
                    manager) in x

    $f045	RECEIVE	receives a message.
    		(carry=1): waits for any message
    		(carry=0): returns immediately, with an error if no message
    		received. Otherwise if a message is received,
    		return sender task id in x, length of
    		data in PCBUF in y (0 means 256) and the optional message
    		type as given with SEND in a.



* * *



    $f048	SETSIG	sets the signal address and the signal mask
    		(carry=1): sets signal address in a/y
    		(carry=0): sets signal mask
    		returns the old mask resp. the old signal address
    		in the same registers.

    $f04b	SENDSIG	c=1: ac contains the signal mask to be sent to the task ID
    		given in xr
    		c=0: set thread to sleep until it receives a signal



Signal handling works as follows: With SETSIG the signal address and the
signal mask can be set. There is only one signal address, that is the address
of a subroutine that is called on a signal. When the routine is called, it has
the current pending signal mask in AC. The routine must return with a code
sniplet



    	pla
    	rti


which jumps directly to the code executed before the signal occured. Note that
the x and y register must not be changed or must be preserved during the
signal routine. The current thread execution is changed to run the signal
handler and to immediately return with the code like above.

SENDSIG can be called from other tasks or from devices, not from a thread in
the same task. When SENDSIG is called, first the signal mask is checked to see
if the signal is allowed. If it is not, SENDSIG just returns. If the signal is
allowed, threads in the receiving task can be in different states: Ready,
Interrupted, or Waiting. If ready or interrupted, the thread is directly set
up to execute the signal routine. If the thread is waiting, it depends on the
INT bit in the signal mask what happens.

If the INT bit is set, then any system call that lets the thread block (i.e.
wait for Semaphore, wait for Receive, wait for Send) will be interrupted. They
then return E_INT in ac - after the signal routine has been called. The other
registers are bogus after an interrupted system call.

If the INT bit is not set, then the signal is pending, and the signal routine
is called when a thread of the task is set to running again.

Signal mask values are



    	SIG_INT		%10000000	/* allow interruptable kernel calls */
    	SIG_CHLD	%01000000	/* Child process has terminated */
    	SIG_TERM	%00100000	/* Child process has terminated */

    	SIG_USR4	%00001000	/* user signal 4 - used for STDLIB */
    	SIG_USR3	%00000100	/* user signal 3 */
    	SIG_USR2	%00000010	/* user signal 2 */
    	SIG_USR1	%00000001	/* user signal 1 */


When a SIG_CHLD is executed, the receiving task is notified that a child has
died. This means it should call CHECKCHLD to see which child has died with
what error code.

The SIG_TERM code should be used by "standard" libraries. Currently there is
no way to inform a task about its death and let it clean up all resources -
that are not bookkept by the kernel! - before it dies. So other libraries
should, instead of calling "KILL", send a SIG_TERM signal when they want to
kill a process. The signal routine of the receiving process should catch this
signal and clean up and then call TERM itself.

SENDSIG with carry clear (wait for signal) must be called by threads only, not
by devices. Also the SIG_INT bit must be set before this call is done. If
there is only one thread in the task, then the thread is sleeping until a
signal is received. When a signal is received, the signal handler is called,
then the thread is awakened and returns with `E_INT`. The other registers are
not preserved.

* * *



    $f04e	TDUP	register a task for a (negative) system task number.
    		parameter: (negative) task number to replace in x,
    		new task number to be used instead in a

    $f051	XRECEIVE receives a message from a specified task only. parameter as
    		with RECEIVE, plus sender task id in x

    $f054	SETNMI	in systems without MMU, set the system NMI routine address.
    		(carry=1): set the new NMI routine address (in a/y)
    			   returns the old NMI address in a/y.
    			   The old address is saved in the place pointed
    			   to by (newadress-4) (see below) by this routine.
    		(carry=0): clear NMI routine.
    		The NMI routine has to save all registers by itself!

    		At NMI address -2 there must be the address of a ctrl routine
    		that can be called via CTRLNMI. Expect that the
    		ctrlnmi routine is called during this SETNMI  with either
    		NMI_ON of NMI_OFF in AC, to set the initial state.
    		The ctrlnmi routines is called with JSR, i.e. return with RTS.

    		At NMI address -4 is a pointer to a memory location
    		where the address of the next NMI routine can be found.
    		If the address found there is $ffff, then the list is at
    		its end.

    		A possible scenario then is:

    		Task calls SETNMI with NEWNMI in a/y. We then have the
    		structure:

    		(NEWNMI-4)	address of oldvec -->   address of OLDNMI
    		(NEWNMI-2)	address of nmictrl
    		NEWNMI -->  	NMI routine code

    		This way the structure can be held in ROM, and still allow
    		flexible NMI chaining. If CTRLNMI is called, each of the
    		nmictrl routines is called with the given value in AC.

    $f057	CTRLNMI	Send command in AC to all currently chained NMI ctrl
    		routines as described with SETNMI. Possible values are:

    		NMI_ON  = allow NMIs
    		NMI_OFF = disallow NMIs

    		This routine is called when initiating and ending an
    		IEC serial bus transfer, for example.



* * *



    $f05a	GETPID	returns the current task ID in x and the current thread ID
    		in y

    $f05d	SLOCK	c=1: locks scheduler to actual task.
    		c=0: unlocks scheduler
    		returns c=0 when ok, c=1 with E_NOTIMP if error.

    		May not be available in all OS implementations.

    $f060	RENICE	changes the priority of the current task.
    		ac = value to add to the priority.
    		returns ac = old priority.

    		May not be available in all OS implementations.

    $f063	CHECKCHLD
                    returns:
    		c=0: returns a child process ID in x and child return code
    		in a of a child process that has died. Only works
    		if the SIGCHLD signal mask has been set when the
    		child died.
    		c=1: no child that has died found.


* * *

#### Error Codes

On bootup the system is tested and possible errors are detected. On systems
with a system port, the hardware error is shown by the number of flashes the
LED makes before the system reboots



    /*        Hardware-Errors          */

    #define   HE_ZP     <-1	/* zeropage mem test */
    #define   HE_RAM    <-2	/* RAM test  (to few RAM) */
    #define   HE_ROM    <-3	/* Couldn't start a program */
    #define   HE_DEV    <-4	/* device returns error upon init */
    #define   HE_TASK   <-5	/* all programs have terminated - reboot */



possible error codes are (from oadef/oa1str.def, where most things, except
filesystem stuff are defined) (These codes must be reordered):




    /*        Software-Errors          */

    #define   E_OK          0       /* no error                             */
    #define   E_NOTIMP      <-1     /* feature/function not implemented     */
    #define   E_CONFIG      <-2     /* in this configuration not available  */
    #define   E_ILLPAR      <-3     /* illegal parameter                    */
    #define   E_NOMEM       <-4     /* no more memory                       */

    /* stream handling errors */
    #define   E_NOSTR       <-5     /* no more streams available            */
    #define   E_SFULL       <-6     /* stream is full                       */
    #define   E_SEMPTY      <-7     /* stream is empty                      */
    #define   E_SLWM        <-8     /* stream below low-water-mark (1/4)    */
    #define   E_SHWM        <-9     /* stream above high-water-mark (3/4)   */
    #define   E_EOF         <-10    /* last byte read, other side has closed */
    #define   E_NUL         <-11    /* noone listening on stream            */

    /* device handling errors */
    #define   E_NODEV       <-12    /* illegal device number                */
    #define   E_DON         <-13    /* device already in use                */
    #define   E_DOFF        <-14    /* device not in use                    */
    #define   E_NOTX        <-15    /* device doesn't send                  */

    /* misc errors */

    #define   E_NOENV       <-16    /* no more free environment/task ID     */
    #define   E_NOSEM       <-17    /* no more free semaphore               */
    #define   E_SEMSET      <-18    /* semaphore is already set (with PSEM) */

    #define   E_NOIRQ       <-19    /* irq routine has not removed irq source */
    #define   E_VERSION     <-20    /* wrong (executable file) version      */

    #define   E_NOTASK      <-21    /* no more free task */
    #define   E_INT         <-22    /* interrupted (by signal) system call */

    #define   E_ILLSIG      <-23    /* illegal signal number                */

    #define   E_TRYAGAIN    <-24    /* try again                            */

    /* file handling errors */

    #define   E_FNODRV      <-32    /* illegal drive number                 */
    #define   E_FNOPATH     <-33    /* wrong path                           */
    #define   E_FILLNAM     <-34    /* illegal name (joker "*","?","\"")    */
    #define   E_FNAMLEN     <-35    /* name too long                        */
    #define   E_FNOFIL      <-36    /* file not found                       */
    #define   E_FWPROT      <-37    /* file write protected                 */
    #define   E_FILEXIST    <-38    /* file exists                          */
    #define   E_FDISKFULL   <-39    /* disk full                            */
    #define   E_FDNEMPTY    <-40    /* subdirectory not empty when rmdir    */
    #define   E_FLOCKED     <-41    /* file locked                          */
    #define   E_FMEDIA      <-42    /* media error                          */
    #define   E_FLOGICAL    <-43    /* bad logical structure on media       */
    #define   E_FINTERNAL   <-44    /* internal error - should not happen!  */

    /* lib6502 errors */

    #define   E_ILLADDR     <-64    /* illegal address for lib6502 mfree    */
    #define   E_NOFILE      <-65    /* illegal file number for lib6502      */
    #define   E_NOSEEK      <-66    /* file not seekable                    */
    #define   E_NOREAD      <-67    /* read on file would block             */
    #define   E_NOWRITE     <-68    /* write on file would block            */
    #define   E_FVERSION    <-69    /* file version number not supported    */


    #define   E_LASTERR     <-96    /* for customized error numbers         */



* * *

#### ROM bootup

During bootup, the kernel initializes the memory and sets up the standard
system memory configuration. This includes mapping the system ROM from the top
of memory down, and the system RAM is mapped from the lower end up.

When the system initialization is done, the ROM is searched for executable
files. The first two bytes of the ROM are taken as an address. If this address
is not `$ffff`, then directly after the first two bytes follows the program
ROM structure described below. After starting the first file, the address at
the beginning is taken as a pointer to the next file and so forth, until the
kernel finds a `$ffff`.

The program ROM structure looks like:



    	P_KIND		0	/* byte, type of file			*/
    	P_ADDR		1	/* word, start address of execution	*/
    	P_RAM		3	/* byte, amount of RAM needed from
    				   lower end of memory in 256-byte
    				   blocks				*/
    	P_SHARED	4	/* byte, size of memory shared with
    				   parent counted from top of memory,
    				   in 256-byte blocks			*/
    	P_PRIORITY	5	/* byte, priority of the task to be
    				   started				*/
    	P_DEV		6	/* 2 byte, the first byte gives the
    				   device number that is opened for the
    				   stdin stream, the second gives the
    				   device number for stdout/stderr	*/
    	P_NAME		8	/* name of the program, ended by
    				   nullbyte. After the nullbyte is the
    				   commandline the program is started
    				   with. The parameter are separated
                                       by nullbytes. After the last parameter
                                       follow two nullbytes to indicate
                                       the end of the command line. */


There are several program types available:



    	PK_PRG		0	/* simple program 			*/
    	PK_DEV		1	/* device block				*/
    	PK_FS		2	/* filesystem				*/
    	PK_INIT		3	/* ROM init process			*/


These file types have to be filled into the `P_KIND` field above. In addition
they can be or'd by the following flags.



    	PK_AUTOEXEC	$80	/* autoexecution 			*/
    	PK_RESTART	$40	/* restart when died			*/


When the `PK_AUTOSTART` bit is not set, the file is ignored by the kernel.

The ROM only starts device blocks (`PK_DEV`) and init processes (`PK_INIT`),
because they are set up before the scheduler starts and therefore the `PCBUF`
is (on non-MMU systems) not available. I.e. init processes do not check their
command line, and do not release the `PCBUF`. Also `PK_INIT` tasks get no
stdin, stdout and stderr streams and have to set them up themselves.

When starting device blocks, the address given in the `P_ADDR` field is given
to the `DEVCMD` command to register the devices.

When using the special init process (init v1.0 from the sysapps directory),
the other entries are started by the init process. `PK_PRG` tasks are started
with their stdin/out/err opened to the devices given by the `PK_DEV` field.
The name and command line are copied from the `P_NAME` field to the
`FORK_NAME` field for the fork.

`PK_FS` are started as programs, but with the stdin/out/err streams set to
`STDNUL`, which is ignored.

If the init process is configured for it, and the `PK_RESTART` bit is set, it
registers the task id of the started process and restarts it when the task
dies.

#### Variable allocation

ROM tasks are started from the `init` task and have no way to determine which
RAM locations are used by other tasks. Therefore the variable allocation must
be done when the ROM is assembled. When using an MMU-based system, where each
task has its own memory environment there need not be much caution. But when
all tasks use the same environment (like in the C64) care has to be taken not
to use a memory location twice. Especially shared libraries must use an own
memory range, and they must be **thread-save**. This allows for different
threads of one task as well as threads of different tasks to use the same
library in one memory environment.

For each environment, task and thread the kernel saves two bytes that are
guaranteed to be unique within the environment, task or thread respectively.
The bytes at addresses 2 and 3 are set to zero when a memory environment is
created and are unique to this environment. The bytes at addresses 4 and 5 are
unique to the task and can be used for task-specific data, like a pointer to a
task structure. The bytes at addresses 6 and 7 are unique to the thread and
can be used for thread data.

* * *

#### Configuration when building

The kernel is configurable in a wide range. Many options are available for all
versions, but also many options are architecture dependent. The options are
generally set in a global file that builds a system ROM. Therefore, to not set
the options in the kernel file itself, the macro `ROM` must be defined.

General options:



    	CMOSCPU		/* have (rockwell) R65C02		*/
    	NMIDEV		/* allows handling of NMI in devices
    			   (only for non-MMU systems, untested)	*/
    	NMIRESET	/* calls RESET when an NMI arrives	*/

    	STACKCOPY	/* on non-MMU systems the stack can
    			   either been split up or, with this
    			   options, copied to a save area	*/


Memory options:



    	RAMSIZE		/* size of checked RAM during bootup,
    			   in 256-byte blocks			*/
    	MIN_MEM		/* minimum memory size to boot		*/
    	RAMTEST		/* if set, RAM is tested, if not RAMSIZE
    			   is taken to be the real RAM size	*/
    	BATMEM		/* memory is not cleared but kept during
    			   memory test				*/
    	NOMIRRORMEM	/* can be set if we _know_ that we do
    			   not have mirrored memory		*/
    	MEMINIVAL	/* value with which to set the memory at
    			   boot. If not set, 0			*/


Some Architecture specific options:



    	NOSYSPORT	/* The CS/A65 computer has a certain 	*/
    			   port where it can read the IRQ line
    			   for example. Not available if set	*/
    	ROMTEST		/* on CS/A65 systems allow booting from
    			   a running OS/A65 (replacing it)	*/


Kernel configuration options for various system calls are the following. The
second half is mostly used to customize for embedded applications.



    	NEED_CHECKCHLD	/* checkchld available if set 		*/
    	NEED_GETINFO	/* getinfo available if set		*/
    	NEED_RENICE	/* renice available if set		*/
    	NEED_SLOCK	/* slock available if set		*/

    	NO_STREAMS	/* streams not available if set		*/
    	NO_FSM		/* filesystem manager not avail. if set	*/
    	NO_SEM		/* semaphores not available if set	*/
    	NO_SEND		/* no send/receive/xreceive if set	*/

    	EOK_SEPARATE	/* kernel is not at end of ROM - 	*/
    			/* kernel/end.a65 is included extra.	*/
    	EOK_NOFILL	/* don't fill between kernel and end of */
    			/* it. kinit _must_ copy the stuff from	*/
    			/* eok_addr to $fff*			*/


* * *

#### Porting

Porting has been made much easier with this kernel version. In general, there
is a `proto` architecture subdirectory that holds some prototype files for a
non-MMU system. that can easily be changed to fit own needs.

The kernel needs two files in the arch/*/kernel subdirectory, namely
`kenv.a65` and `kinit.a65`. The `kinit.a65` file is included in the kernel
startup code, i.e. after setting the interrupt flag and clearing the decimal
flag this file is executed, and it is expected that the CPU gets out of this
file at the end of it.

##### kinit.a65

This file must initialize the stack, the memory management and mapping of the
system ROM and RAM. Then the system preemption timer must be set up. When
including `kernel/zerotest.a65` and `kernel/ramtest.a65` here all the memory
options from above are available. Also the ramtest file returns the size of
memory found in 256-byte blocks in a. This value must be passed to the end of
the init routine in this file.

In addition to this some Macros have to be defined, that are used in the
kernel later.



      	GETFREQ()	returns a=0 for 1Mhz, a=128 for 2MHz
      	LEDPORT		address of LED port to signal hardware failures
      	LED_LED		bit of LED bit in LEDPORT address
      	H_ERROR()	if LEDPORT is not defined, then H_ERROR replaces
        	                the LED toggle routine if needed.
      	CLRTIMER()	clear the system preemption timer interrupt
      	KERNELIRQ()	returns if an interrupt occured in the kernel
         	                or outside. Is checked by "BNE IRQ_in_Kernel"
      	SAVEMEM()	save system mem config for interrupt
      	RESTOREMEM()	restore system mem config after interrupt
      	STARTMEM()	This is used to alloc/enable memory pages found
         	                during kernel init. Takes mem size in AC as given
       	                by the kernel init. It is called _after_ inimem!


For the most systems without MMU, most of these macros can be taken from the
C64 `kinit.a65` file. If you have a more complicated mapping, you can have a
look at the CS/A65 `kinit.a65` file.

##### kenv.a65

The `kenv.a65` file is included in another part of the kernel. It is the part
that maps the memory according to the environment number for each task.
Several routines must be defined here:



    	inienv		init environment handling
    	setthread	set active thread
    	initsp		init task for thread (no in xr)
    	push		push a byte to active threads stack
    	pull		pull a byte from active threads stack
    	memtask		jump to active thread
    	memsys		enter kernel

    	getenv		get e free environment
    	freenv		free an environmen
    	sbrk		reduce/increase process memory in an environment

    	enmem		enable memory blocks for environments
    			(this kernel call is used for MMU systems and
    			heavily system specific)
    	setblk		sets a specific memory block in the memory map
    			of the active task. Also for MMU systems only


Also some macros must be defined:



    	MEMTASK2()	This is equivalent to memtask above, but it
    			is used for returns from interrupt
    	MAPSYSBUF	maps the PCBUF of the active task to the
    			address given by SYSBUF in system memory
    	SYSBUF		mapped tasks PCBUF
    	MAPENV()	maps the address given in a/y in env x to somewhere
    			in the kernel map, returns mapped address in a/y
    	MAPAENV()	as MAPENV, but does it for actual env.
    	GETTASKMEM	returns (in AC) how much memory a task has
    			(task id in y)
    	CPPCBUFRX	copies PCBUF from other task (yr) to active one
    	CPPCBUFTX	copies PCBUF from active task to another one (yr)
    	CPFORKBUF()	copies the PCBUF from the FORK call to the task.
    	GETACTENV()	returns active environment in ac


The full descriptions can be found in the C64 kenv file for example. This may
look complicated, but it isn't. If you have a simple 6502 system without
memory management, just copy (or set a link) to the C64 file. If you have a
more complicated system, have a look at the CS/A65 file.

* * *

#### System Constants

Terminal devices should understand the following terminal control codes (But
the currently implemented device driver doesn't understand them all,
though...)



    /*        Terminal Commands        */

    #define   TC_BEL    7		bell
    #define   TC_BS     8		backspace
    #define   TC_HT     9		horizontal tabulator
    #define   TC_LF     10		line feed
    #define   TC_VT     11		vertical tabulator
    #define   TC_FF     12		form feed
    #define   TC_CR     13		carriage return
    #define   TC_ESC    27		Escape code

    #define   TC_CLFT   $80		cursor left
    #define   TC_CRGT   $81		cursor right
    #define   TC_CUP    $82		cursor up
    #define   TC_CDWN   $83		cursor down
    #define   TC_HOME   $84		cursor to the upper left edge
    #define   TC_CLR    $85		clear screen
    #define   TC_DEL    $86		delete char
    #define   TC_INS    $87		insert
    #define   TC_WLO    $88    	set upper left window corner by cursor pos
    #define   TC_WRU    $89  	set lower right window corner by cursor pos
    #define   TC_WCLS   $8a		clear window
    #define   TC_EOL    $8b		put cursor to the end of line
    #define   TC_CLL    $8c		clear rest of line from cursor

    #define   TC_ECHO   $8d		switch on local (device) echo mode
    #define   TC_NOECHO $8e		device only sends, application has to echo

    #define   TC_CPOS   $8f         next two chars are row and column of new
                                    cursor position



FORK, SETIRQ and TRESET can **NO MORE** be called via a SEND system call, when
the receiver address is SEND_SYS and the message type is SP_* !!



    /*        SysProcCalls             */

    #define   PCBUF     $200



STD* stream number are replaced by the numbers saved in the environment
struct.




    /*        StdStream                */

    #define   STDNUL         $fc       /* will be ignored (for FS STDIO)*/
    #define   STDIN          $fd
    #define   STDOUT         $fe
    #define   STDERR         $ff



Reserved system environment numbers:




    #define   SEND_FM        $fe		/* filesystem manager */

    #define	  SEND_ERROR	 $fd		/* critical error handler */

    #define   SEND_TIME      $fc		/* set/get actual time */

    #define   SEND_NET       $fb            /* open network connections */
