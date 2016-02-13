# OneTouch Verio 2015

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.eu).

The communication protocol as described in this document applies to
the following models:

 * *OneTouch Verio 2015* (microUSB connector)
 * *OneTouch Select Plus*

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

No knock sequence is needed to initiate communication.

## Identification

Since the device communicates by what would otherwise be a destructive
change to its storage, it is important that the commands are not
issued to a non-compatible device, as they might damage the partition
table of an external storage device.

It is possible to identify the device by inspecting either the USB
Device descriptor, or issuing a SCSI **INQUIRY** command.

### USB identification

In the USB descriptor, the device will report an *Interface
descriptor* as such:

    bInterfaceClass         8 Mass Storage
    bInterfaceSubClass      6 SCSI
    bInterfaceProtocol     80 Bulk-Only
    iInterface              7 LifeScan MSC

The `LifeScan MSC` string identifies the mass-storage controller
protocol as being non-standard.

The USB `iProduct` and `iSerial` are also matching the information
reported by the LifeScan communication protocol.

### SCSI identification

At the SCSI level, the device will report a *Vendor identification*
string of `LifeScan`:

    Vendor identification: LifeScan
    Product identification:
    Product revision level:
    Unit serial number:

Note that while the USB protocol information (product, serial number)
match the device's information, only the vendor identification is
visible in response to the **INQUIRY** command.

## Message structure

Message, both sent and received, in any of the registers follow the
same structure:

    message = STX length message ETX checksum
    STX = %x02
    length = 2OCTET             ; 16-bit little-endian value
    message = [length - 6]OCTET
    ETX = %x03
    checksum = 2OCTET           ; 16-bit little-endian value

The messages are variable length, with the length provided by the
16-bit word following the `STX` constant. The value of length includes
the STX/ETX constants, the length itself and the checksum, leaving the
body of the message 6 bytes short.

While no command has been recorded using more than 255 bytes, the
length is assumed to be 16-bit to match out of band information in the
original software's debug logs. The size of the register also makes
the maximum lenght of the command 512 bytes (0x200).

The `STX` and `ETX` constants to mark start and end of the message
match the constants used in the OneTouch Ultra Easy serial protocol,
and are thus named the same way as in its specs.

The `checksum` is a variant of CRC-16-CCITT, seeded at `0xFFFF`, and
stored little-endian, again the same as used in the OneTouch Ultra
Easy protocol.

## Messages

Messages are binary, and only some are related to each other in any
obvious way.

The commands have been named after their function, in the style of
SCSI commands:

 * **QUERY** to retrieve information on the device (serial number,
   device model, etc.)
 * **READ RTC** to retrieve current RTC time of the device.
 * **WRITE RTC** to change the RTC time fo the device.
 * **READ RECORD COUNT** to retrieve the number of records in the
   device's memory.
 * **READ RECORD** to retrieve the content of one record.

### QUERY

A single message with a byte specification provides information on the
hardware device. The request is sent through, and the response read
from, `lba3`.

    QUERY-request = STX %x0a %x00 ; message length = 10 bytes
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

    QUERY-response = STX length
                     %x04 %x06 *WCHAR-BE
                     ETX checksum

It is interesting to note the usage of big endian wide characters, as
the rest of the protocol is little endian. This is probably due to
Unicode specifying BE being the default.

### READ RTC

The request to query the device time is fairly simple, and is
communicated through `lba3`:

    READ RTC-request = STX %x09 %x00 ; message length = 9 bytes
                       %x04 %x20 %x02
                       ETX checksum

    READ RTC-response = STX %x0c %x00 ; message length = 12 bytes
                        %x04 %x06 timestamp
                        ETX checksum

    timestamp = 4OCTET ; 32-bit little-endian value

#### Timestamp format

Timestamp, both for the device's clock and for the reading records, is
defined as a little-endian 32-bit number, representing the number of
seconds since **2000-01-01 00:00:00**.

It should not be mistaken for a UNIX timestamp, although the format is
compatible. To convert to UNIX timestamp, you should add `946684800`
to the value (the UNIX timestamp of the device's own epoch.)

### WRITE RTC

The **WRITE RTC** command is also communicated through `lba3`.

    WRITE RTC-request = STX %x0d %x00 ; message length = 13 bytes
                        %x04 %x20 %x01 timestamp
                        ETX checksum

    WRITE RTC-response = STX %x08 %x00 ; message length = 8 bytes
                         %x04 %x06
                         ETX checksum

### READ RECORD COUNT

The following messages correspond to request and response for the
number of records in memory. The messages are transmitted over `lba3`.

    READ-RECORD-COUNT-request = STX %x09 %x00 ; message length = 9 bytes
                                %x04 %x27 %x00
                                ETX checksum

    READ-RECORD-COUNT-response = STX %x0a %x00 ; message length = 10 bytes
                                 %x04 %x06 message-count
                                 ETX checksum
    message-count = 2OCTET ; 16-bit little-endian value

### READ RECORD

The records are then accessed through indexes between 0 and
`message-count` (excluded) as reported by READ RECORD COUNT.

    READ-RECORD-request = STX %x0c %x00 ; message length = 12 bytes
                          %x04 %x31 %x02 record-number %x00
                          ETX checksum
    record-number = 2OCTET ; 16-bit little-endian value

The record number is assumed to be a 16-bit little endian value, as
the Verio 2015 is reported to store up to 500 results. It is also
consistent with the UltraEasy protocol.

Records are stored in descending time order, which means record `0` is
the latest reading.

    READ-RECORD-response = STX %x18 %x00 ; message length = 24 bytes
                           %x04 %x06 inverse-record-number
                           %x00 unknown-counter
                           timestamp glucose-value flags %x0b %x00
                           ETX checksum

    inverse-record-number = 2OCTET ; 16-bit little-endian value
    unknown-counter = 2OCTET       ; 16-bit little-endian value
    glucose-value = 4OCTET         ; 32-bit little-endian value
    flags = OCTET

The inverse record number seem to provide a sequence of readings, it
would be interesting to compare its value for a reader that exceeded
its storage memory.

An unknown counter, also proceeding backward is present, which is
offset to the inverse record number. Different meters appear to have
different offsets.

The glucose value is represented as a 32-bit little endian value, to
continue the similarities with the UltraEasy device. It represent the
blood sugar in mg/dL. As most other meters, the eventual conversion to
mmol/L happens only at display time.

The flags are not currently well understood; the device allows setting
comments on the readings, but these responses are not visible from the
interface of the device itself; a "speech bubble" appears, steady or
blinking, on some of the readings instead. The original software does
not seem to expose the data correctly either.

At least two information are likely to be found in these flags, or in
the following constant `0x0b` byte: the type of measurement (plasma v
whole blood) and the measurement site (fingertip), as those are
visible in the original software's UI. Meal information might also be
present. OneTouch Ultra2 provided a simple mapping of meal and comment
codes.
