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

### USB IDs

| Device               | Vendor ID | Product ID |
| ---                  | ---       | ---        |
| OneTouch Verio 2015  | 2766      | 0000       |
| OneTouch Select Plus | 2766      | 1000       |

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

There does not appear to be any specific requirement of timing between
the **WRITE(10)** command the subsequent **READ(10)**. The response
for a given request is read from the same register it has been written
to. Message are specific to one register and will not work if written
to a different register.

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

## Packet Structure

The registers are all 512 bytes in size (SCSI block size), and contain a padded
packet as defined by the [Shared Binary Protocol](shared-binary-protocol.md).

Within the packet, the `link-control` byte is unused and always left at
`0x00`.

The `command-prefix` byte may have one of accepted values as defined below, but
for ease of implementation, `0x03` is suggested, as the **READ PARAMETER**
command will not echo the chosen prefix, but always return `0x03`.

    packet                  ; see shared-binary-protocol.md
    link-control = %x00     ; not used by this device
    command-prefix = %x03 / ; suggested
                     %x04 / %x05
## Timestamp Format

Timestamps, both for the device's clock and for the reading records, are defined
as a little-endian 32-bit number, representing the number of seconds since
**2000-01-01 00:00:00**.

It should not be mistaken for a UNIX timestamp, although the format is
compatible. To convert to UNIX timestamp, you should add `946684800` to the
value (the UNIX timestamp of the device's own epoch.)

## Messages

Messages are binary, and only some are related to each other in any
obvious way.

The commands have been named after their function, in the style of
SCSI commands:

 * **QUERY** to retrieve information on the device (serial number,
   device model, etc.)
 * **READ PARAMETER** to retrieve parameters set for the device (time
   and date format, glucose unit used.)
 * **READ RTC** to retrieve current RTC time of the device.
 * **WRITE RTC** to change the RTC time fo the device.
 * **READ RECORD COUNT** to retrieve the number of records in the
   device's memory.
 * **READ RECORD** to retrieve the content of one record.
 * **ERASE MEMORY** to clear the meter altogether.

### QUERY

A single message with a byte specification provides information on the
hardware device. The request is sent through, and the response read
from, `lba3`.

    QUERY-request = STX %x0a %x00 ; message length = 10 bytes
                    %x03 %xE6 %x02 query-selector
                    ETX checksum

    query-selector = query-selector-serial /
                     query-selector-model /
                     query-selector-software /
                     query-selector-unknown /
                     query-selector-date-format /
                     query-selector-time-format /
                     query-selector-url /
                     query-selector-languages
    query-selector-serial      = %x00
    query-selector-model       = %x01
    query-selector-software    = %x02
    query-selector-unknown     = %x03
    query-selector-date-format = %x04
    query-selector-time-format = %x05
    query-selector-url         = %x07  ; http://www.lifescan.co.uk
    query-selector-languages   = %x09

The reply starts with what appears an arbitrary pair of bytes, and
then follows with what appears to be a UTF-16-LE string, NULL
terminated (except for the `query-selector-unknown` response.)

    QUERY-response = STX length
                     %x03 %x06 *WCHAR-LE %x00 %x00
                     ETX checksum

#### Languages

Devices sold on multilingual markets allow the selection of the
language to use through settings. The **QUERY** selector `0x09`
appears to provide the list of supported languages in the device.

    WALPHA-UPPER = %x41-5A %x00
    WALPHA-LOWER = %x61-7a %x00
    WDIGIT = %x30-39 %x00
    WIDESP = SP %x00
    WSEMICOLON = ";" %x00
    WDOT = "." %x00

    LANGUAGES = language *(WSEMICOLON WIDESP language)
    language = language-code country-specifier WIDESP language-version
    language-code = 2WALPHA-UPPER
    country-specifier = WALPHA-LOWER
    language-version = 2WDIGIT WDOT 2WDIGIT WDOT 2WDIGIT


The languages are given as a list of wide-char (little endian)
specifications of languages. Each language specification includes a
main language code (which appears to match ISO language codes) and
some country specification: `ENu` for *English (US)* and `ENe` for
*English (UK)*.


#### Date and time format

The device appears to provide some information on the date and time
format to use for displaying date and time, although this does not
match the actual format displayed on the device.

The format appears to be similar to `strftime`, but it is not
compatible with the POSIX interface for it.

| Specifier | Meaning                               |
| --------- | ------------------------------------- |
|    %I     | Hour as decimal number, 24-hour clock |
|    %h     | Hour as decimal number, 12-hour clock |
|    %T     | Minute as decimal number              |
|    %p     | Either "AM" or "PM"                   |
|    %D     | Day of the month as decimal number    |
|    %n     | Abbreviated month name                |
|    %y     | Year as decimal number                |

### READ PARAMETER

The meter repors a number of parameters in a similar fashion to the
**QUERY** command. The request is sent through, and the response read
from, `lba4`.

    READ-PARAMETER-request = STX %x09 %x00 ; message length = 9 bytes
                             %x03 parameter-selector OCTET
                             ETX checksum

    parameter-selector = parameter-selector-timefmt /
                         parameter-selector-datefmt /
                         parameter-selector-unit

    parameter-selector-timefmt = %x00
    parameter-selector-datefmt = %x02
    parameter-selector-unit = %x04

The `OCTET` following the selector appears to be completely ignored.

The response is provided in different formats, depending on the
parameter requested.

    READ-PARAMETER-response = STX length
                              %x03 %x06 parameter-response
                              ETX checksum

    parameter-response = parameter-response-timefmt /
                         parameter-response-datefmt /
                         parameter-response-unit

    parameter-response-timefmt = *WCHAR-LE %x00 %x00
    parameter-response-datefmt = *WCHAR-LE %x00 %x00

    parameter-response-unit = parameter-unit-mgdl / parameter-unit-mmol
                              %x00 %x00 %x00
    parameter-unit-mgdl = %x00
    parameter-unit-mmoll = %x01

Time and date formats match those returned by the **QUERY** command.

### READ RTC

The request to query the device time is fairly simple, and is
communicated through `lba3`:

    READ-RTC-request = STX %x09 %x00 ; message length = 9 bytes
                       %x03 %x20 %x02
                       ETX checksum

    READ-RTC-response = STX %x0c %x00 ; message length = 12 bytes
                        %x03 %x06 timestamp
                        ETX checksum

    timestamp = 4OCTET ; 32-bit little-endian value

### WRITE RTC

The **WRITE RTC** command is also communicated through `lba3`.

    WRITE-RTC-request = STX %x0d %x00 ; message length = 13 bytes
                        %x03 %x20 %x01 timestamp
                        ETX checksum

    WRITE-RTC-response = STX %x08 %x00 ; message length = 8 bytes
                         %x03 %x06
                         ETX checksum

### READ RECORD COUNT

The following messages correspond to request and response for the
number of records in memory. The messages are transmitted over `lba3`.

    READ-RECORD-COUNT-request = STX %x09 %x00 ; message length = 9 bytes
                                %x03 %x27 %x00
                                ETX checksum

    READ-RECORD-COUNT-response = STX %x0a %x00 ; message length = 10 bytes
                                 %x03 %x06 message-count
                                 ETX checksum
    message-count = 2OCTET ; 16-bit little-endian value

### READ RECORD

The records are then accessed through indexes between 0 and
`message-count` (excluded) as reported by **READ RECORD COUNT**.

    READ-RECORD-request = STX %x0c %x00 ; message length = 12 bytes
                          %x03 %x31 %x02 record-number %x00
                          ETX checksum
    record-number = 2OCTET ; 16-bit little-endian value

The record number is assumed to be a 16-bit little endian value, as the Verio
2015 is reported to store up to 500 results. It is also consistent with
the [OneTouch UltraEasy](onetouch-ultraeasy.md) protocol.

Records are stored in descending time order, which means record `0` is
the latest reading.

    READ-RECORD-response = STX %x18 %x00 ; message length = 24 bytes
                           %x03 %x06 inverse-record-number
                           %x00 lifetime-counter
                           timestamp glucose-value flag-meal %x00
                           other-flags %x0b %x00
                           ETX checksum

    inverse-record-number = 2OCTET ; 16-bit little-endian value
    liftime-counter = 2OCTET       ; 16-bit little-endian value
    glucose-value = 2OCTET         ; 16-bit little-endian value
    other-flags = OCTET

    flag-meal = meal-none / meal-before / meal-after
    meal-none = %x00
    meal-before = %x01
    meal-after = %x02

The inverse record number seem to provide a sequence of readings, it
would be interesting to compare its value for a reader that exceeded
its storage memory.

A lifetime counter is also present, that will keep increasing even
though the device's memory is cleared with the **ERASE MEMORY**
command. The original offset of the meter is likely related to the
factory calibration.

The glucose value is represented as a 16-bit little endian value. It represent
the blood sugar in mg/dL. As most other meters, the eventual conversion to
mmol/L happens only at display time.

The meal flag is a single byte, representing a tristate of no information,
before meal reading, and after meal reading. This meal information cannot be set
on the Verio meter, but can be set on the Select Plus.

The second set of flags are not currently well understood; Verio meters allow
setting comments on the readings, but these responses are not visible from the
interface of the device itself; a "speech bubble" appears, steady or blinking,
on some of the readings instead. The original software does not seem to expose
the data correctly either. These may be used by other models such as Select
Plus.

At least two information are likely to be found in these flags, or in the
following constant `0x0b` byte: the type of measurement (plasma v whole blood)
and the measurement site (fingertip), as those are visible in the original
software's UI.

*Caution:* at least one byte in the record will likely represent a control
solution test, rather than an actual blood glucose reading; which value is that
is still unknown.

### MEMORY ERASE

The memory erase command deletes all the records in the device's
memory. It is communicated over `lba3`.

    MEMORY-ERASE-request = STX %x08 %x00 ; message length = 8 bytes
                           %x03 %x1a
                           ETX checksum

    MEMORY-ERASE-response = STX %x08 %x00 ; message length = 8 bytes
                            %x03 %x06
                            ETX checksum

Remember that this action is irreversible.
