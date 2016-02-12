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
and 5. These will be referenced as `lba3`, `lba4` and `lba5` in this
text.

The registers are accessed through SCSI commands **WRITE(10)** and
**READ(10)**. Raw commands seem to be required, as the device rejects
any CDB (Command Block) with non-default flags set.

## Message structure

Message, both sent and received, in any of the registers follow the
same structure:

    const uint8_t STX = 0x02;
    uint16_t length; // little endian
    char[length-6] message; // length includes preamble, length and checksum
    const uint8_t ETX = 0x03;
    le_uint16_t checksum; // CRC-CCITT 0xFFFF

The messages are variable length, with the length stated in the second
byte of the message; `length` in this case is calculated until the end
of the checksum. Even though length appears to be 16-bit in length,
none of the known messages appear to take more than a single byte
length. The register size also limits the maximum length to `0x100`.

The `STX` and `ETX` constants to mark start and end of the message
match the constants used in the OneTouch Ultra Easy serial protocol,
and are thus named the same way as in its specs.

The checksum itself is a variant of CRC-16-CCITT, seeded at `0xFFFF`,
and stored little-endian, again the same as used in the OneTouch Ultra
Easy protocol.

## Messages

Messages are binary, and only some are related to each other in any
obvious way.

### Information query (serial, software, …)

A single message with a byte specification provides information on the
hardware device. The request is sent through, and the response read
from, `lba03`.

    query-request = STX %x0a %x00 ; message length = 10 bytes
                    %x04 %xE6 %x02 query-selector
                    ETX checksum

    query-selector = query-selector-serial /
                     query-selector-model /
                     query-selector-software /
                     query-selector-unknown
    query-selector-serial = %x00
    query-selector-model = %x01
    query-selector-software = %x02
    query-selector-unknown = %x03

The reply starts with what appears an arbitrary pair of bytes, and
then follows with what appears to be a UTF-16-BE string, null
terminated (except for the `query-selector-unknown` response.)

    query-response = STX length
                     %x04 %x06 *WCHAR-BE
                     ETX checksum

It is interesting to note the usage of big endian wide characters, as
the rest of the protocol is little endian. This is probably due to
Unicode specifying BE being the default.

## Date/time format

The protocol define a point in time (for the current device clock, or
the timestamp of a reading), as a little-endian four-bytes count of
seconds since **01/01/2000 @ 00:00**.

This differs from the otherwise similar UltraEasy protocol, that uses
the UNIX Epoch of **01/01/1970 @ 00:00**.
