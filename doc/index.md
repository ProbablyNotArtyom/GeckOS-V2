# OS/A65 Operating System

## Version 2.0.1

## (c) 1989-1998 Andre Fachat [[Homepage]](http://www.tu-chemnitz.de/~fachat/index.html)

* * *

This is a completely new version of my 6502 operating system GeckOS/A65. From
version 2.0.0 on it has a lot of new features:

  * multithreading
  * dynamic memory management
  * relocatable fileformat
  * lib6502 standard library
  * internet support

### Description

OS/A65 is a full-featured Multitasking/Multithreading operating system for the
6502. It is preemptive and implements some Unix-like features, like signals,
semaphores, [relocatable fileformat](fileformat.md), [standard
library](lib6502.md), internet support via a kind of simplified sockets and
last but not least virtual consoles.

It is extremly scalable. Stripped down to the scheduler and interrupt handling
the kernel is only slightly above 2k. In normal [embedded
systems](embedded.md) the kernel has around 4k, with only application
programs running. Full featured systems have a 4k kernel, and several support
tasks provided system services like TCP/SLIP and (different) filesystems.

The kernel is almost completely hardware independent. All the architecture-
specific stuff is in a separate subdirectory for each architecture.

The [lib6502](lib6502.md) as standard library allows easy access to the
system services. Parts of this library are already implemented in another 6502
operating system, Lunix by Daniel Dallmann. This way source code compatibility
is achieved.

Version 2.0.0 features a "slipd" server process that brings easy internet
access to all lib6502 programs, that can now access TCP connections like
files. A stable WWW server running on the OS is built into the slipd daemon.
Also a remote login can be done. This way the OS can run programs to for
example read sensors and write the stuff to files, which are exported by the
WWW server.

The relocatable o65 fileformat used by the lib6502 standard library in version
2.0.0 allows more than one instance of a program being run at the same time
without interference, even without virtual memory. Also the very same binaries
runs on all supported platforms (if they do not use architecture specific
stuff, but lib6502 calls only).

### Architectures

Architectures supported are the C64, as well as my [CS/A65 MMU](http://www.tu-chemnitz.de/~fachat/8bit/hardware/csa/index.html) selfbuilt computer and my
[CS/A65 Gecko](http://www.tu-chemnitz.de/~fachat/8bit/hardware/gecko/index.html) board. Also supported are
the Commodore CBM8096 and CBM8296 computers, as well as any 32k RAM PET (the
3032, 4032 and 8032)

  * [CBM PET](cbm8x96.md) documentation. Supported are 40 and 80 column models 3032, 4032, 8032, 8096 and 8296.

  * [Embedded systems](embedded.md) need not all features. Here is some doc to strip the OS down to the basics (with around 2.5k in the end...)

### Development

For the development of OS/A65 programs there are two possibilities:

  * [lib6502](lib6502.md) with the [o65](fileformat.md) file format. This allows source compatibility (to some degree) with Lunix, as well as that the program runs on all supported platforms.

lib6502 programs are simply assembled with my xa65 crossassembler with the
including the file "lib6502.i65" and the assembler option "-LLIB6502" set.
This tells the compiler to put "LIB6502" into the file as undefined reference
that is resolved when loading. The lib6502 jump table is relative to this
address.

  * A system application not only uses lib6502 calls (if it uses them) but also [kernel](kernel.md) calls. The kernel can be at different addresses for different architecture as well. Therefore you have to add "-LOSA2KERNEL" to the assembler line. This address is also resolved when loading. If the file should also be used as a ROM file, then it has to have a ROM boot header, see the kernel description.

### More Docs

  * Here is the cross assembler [xa](http://www.tu-chemnitz.de/~fachat/8bit/cross/xa/index.html) you need to assemble the whole stuff

  * What's [new](LOG-2.0) in this version since 2.0.0

  * How to [build](build.md) the binaries

  * A description of the [files](files.txt) in the archive

  * [kernel](kernel.md) interface

  * [lib6502](lib6502.md) description

  * Operation [without an MMU](nommu.md)

  * introduction to the [devices](devices.md)

  * [filesystem](filesystems.md) interface

  * The [README](README) that comes with the binary.

  * The [README.c64](README.c64) with instructions how to run it on the C64

  * The [README.slip](README.slip) with instructions how to run the TCP/SLIP software

  * There also is a list of [Known Bugs](BUGS)

  * There also is a list of [Ideas](IDEAS) what to do next...

  * An instruction to the lib6502 [lsh](README.lsh)

Old stuff:

  * [**Overview**](oa1.md) over the Computer System and its Software

  * The old [standard library](oldlib.md) has been replace with the lib6502 and is not longer supported.

  * The old summary of [shell](shell.md) has been replaced by the lib6502 lsh and will no longer be developed.

  * The old [monitor](mon.md) is still supported, but not actively developed any more.

  * summary of features and extensions of the [BASIC](basic.md) interpreter **(c) Commodore**

  * The [ChangeLog](LOG-1.3) for version 1.3.* and for the development of [2.0.0](LOG-pre-2.0).

### History

I didn't dream of this becoming such a nice project when I started building
[this computer](http://www.tu-
chemnitz.de/~fachat/8bit/hardware/csa/index.html) in 1989.

After someone asked me to release it to the public, I decided to put it under
the [GNU public license](COPYING). (Which, of course, doesn't hold true for
the ported BASIC interpreter, which is taken from the C64. See this
[file](basic.md) for more.) Also the character ROMs are taken from the C64.
However, Commodore in its old form doesn't exist anymore and attempts to
contact the new right holders have not brought any success, so I put them
here.

Well, when I did this project, it was just for fun. But now I find it quite
nice. Well, if you know some magazin that would like to publish some of it, I
will be glad writing an article or so (if anybody really wants it ;-)
But on the other hand my interests have moved. Occasionally I still work on
the project - when I have the time (or take the time ;-)
But after all, I don't really have time for it.

### Ideas for later versions

  * vt100 control codes for the console.

  * native C128 port

  * in this process abstract a kind of block device from fsibm and use it for the VC1571 as well

* * *

Last modified 14 march 1997 by A. Fachat
This Page has been read approx. ![\[Can't display counter!\]](/cgi-
bin/counter/~fachat/8bit/osa/v2.0/index.html) times since march 14, 1996.
