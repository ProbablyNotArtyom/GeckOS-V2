#  OS/A65 CBM PET Notes
#### OS/A65 CBM PET Version 2.0
#### (c) 1998 Andre Fachat

* * *

# CBM 8x96

The Commodore PET 8096 and 8296 (also known as CBM 8096 and CBM 9296)
computers have at least 96 kByte of RAM and can remap the ROM area with RAM.
This makes them a nice base for OS/A65. Although they don't have many I/O
devices at least they provide an 80 columns screen.

#### Memory Map

In the CBM 8x96 one can remap the upper 32k of CPU address space by writing
certain values to $fff0. The blocks `$8000-$bfff` and `$c000-$ffff` can be
remapped to two different banks of RAM each. The screen memory (`$8000-$9000`)
and I/O (`$e800-$efff`) can be set to `peek-through' separately. In the OS/A65
memory configuration the lower 32k and one bank of the upper 32k of RAM are
used for the OS image (the `ROM') and the lib6502 managed application memory.
The screen memory and I/O area are not accessible for lib6502 applications.
The second upper 32k RAM bank is used (with screen and I/O mapped through) as
system memory area, together with some RAM from `$300-$b00`. Also there are
the mirrors of the other virtual console screen areas - when switching screens
they are copied around.


    Addr    | RAM 0             | RAM 1
    ========|===================|==================
    $ffff   | ----------------- | -----------------
    $fff0   |  config register  |  config register
    $ff00   | ----------------- | -----------------
            |                   |   system memory
    $f000   |                   | -----------------
            |                   |        I/O
    $e800   |                   | -----------------
            |  lib6502 memory   |  lib6502 memory
    $b000   |                   | -----------------
            |                   |  virtual screens
    $9000   |                   | -----------------
            |                   |      screen
    $8000   |                   | -----------------
            |                   |
    $6800   | ----------------- |
            | OS/A65 ROM image  |
    $0b00   | ----------------- |
            |    system RAM     |     no memory!
    $0200   | ----------------- |
            |       Stack       |
    $0100   | ----------------- |
            |     Zeropage      |
    $0000   | ----------------- | -----------------


#### ROM image build

Building the 8x96 ROM image is simple. Just go to the arch/cbm8x96/boot
directory and type "make". This should build all files. "make osa.d64" should
make a d64 VC1541 disk image with all necessary files. You have to have my new
XA version in the path, though.

To customize the ROM, change edit rom.a65 in arch/cbm8x96. You can gain some
memory when you remove the shell/monitor from the ROM image. (You could
replace it with the lib6502 monitor "mon" to be loaded like "lsh".) When
running "make" then the command "file65" should print the addresses used for
the different segments. They must not overlap. In addition the data segment
must be below $8000, as it is needed by lib6502 that can be called from both
RAM banks. To move the lib6502 RAM area, edit the values for `Memstart,
Memend` and `Zerostart` at the end of rom.a65. To move the segments, edit the
Makefile and change the addresses for the segments in the XA calls. The text
segment address must be two below the actual start as it also counts the two
byte starting address used for loading. In addition you have to edit the value
for the definition of ROMSTART in rom.a65.

#### Boot

Put the files "loader", "lsh" and "rom" on a disk accessible by the PET 8x96.
load "loader" into BASIC memory and start it (probably edit it for the right
load device number). This boots OS/A65.

* * *

# Any 80 columns PET with 32k

Build a version for this computer is a bit more difficult because of the
restricted memory. I have now tried to build such a version. The device does
not (yet) check for 40 columns but can now use only 80 cols.

#### Memory Map

The memory map is straightforward. First zeropage, stack, then system memory
(all of it), then OS image, then lib6502 memory. At the end ($7800) a second
virtual screen, and at $8000 the normal screen area. At $90-$93 in the
zeropage the PET ROM IRQ and BRK vectors are located and have to be used.

#### Build 'n Boot

Basically the same applies as for the 8x96.
