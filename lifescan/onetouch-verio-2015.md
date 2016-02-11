# OneTouch Verio 2015

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.eu).

## Important device notes

Not to be confused with the the previous generation of OneTouch Verio
meters, the 2015 edition comes with a microUSB-A connector, rather
than a TRS (stereo-jack.)

## Communication

The device is identified by operating systems as a standard USB Mass
Storage device of disk type. No custom drivers are required.

The original software does not use any knocking sequence, no extra
device or interface is available.

Communication is apparently achieved through three 512-bytes
[registers](https://en.wikipedia.org/wiki/Hardware_register),
implemented as flash device sectors with
[LBA](https://en.wikipedia.org/wiki/Logical_block_addressing) 3, 4
and 5.

The registers are accessed through SCSI commands **WRITE(10)** and
**READ(10)**. Raw commands seem to be required, as the device rejects
any CDB (Command Block) with non-default flags set.

## Messages

Messages are not currently understood, further information to follow.
