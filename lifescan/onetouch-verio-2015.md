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
from, `lba3`.

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

### Device time

The request to query the device time is fairly simple, and is
communicated through `lba3`:

    time-request = STX %x09 %x00 ; message length = 9 bytes
                   %x04 %x20 %x02
                   ETX checksum

    time-response = STX %x0c %x00 ; message length = 12 bytes
                    %x04 %x06 timestamp
                    ETX checksum

    timestamp = 4OCTET ; 32-bit little-endian value

See [timestamp format](#timestamp-format) the details of timestamp
handling for this device.

### Record access

Access to the records in the device's memory is done by requesting
them singularly (similar to the UltraEasy protocol). This requires
first querying for the number of records present.

The following messages correspond to request and response for the
number of records in memory. The messages are transmitted over `lba3`.

    record-count-request = STX %x09 %x00 ; message length = 9 bytes
                           %x04 %x27 %x00
                           ETX checksum

    record-count-response = STX %x.. %x00 ; message length =
                            %x04 %x06 message-count
                            ETX checksum
    message-count = 2OCTET ; 16-bit little-endian value

The message IDs are then accessed through indexes between 0 and
`message-count` (excluded):

    read-record-request = STX %x0c %x00 ; message length = 12 bytes
                          %x04 %x31 %x02 record-number %x00
                          ETX checksum
    record-number = 2OCTET ; 16-bit little-endian value

The record number is assumed to be a 16-bit little endian value, even
though this couldn't be confirmed with a device with more than 256
records, it would be consistent with the UltraEasy protocol.

Records are stored in descending time order, which means record `0` is
the latest reading.

    read-record-response = STX %0. %x00 ; message length =
                           %x04 %x06 inverse-record-number %x00 unknown-counter
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

## Timestamp format

The protocol define a point in time (for the current device clock, or
the timestamp of a reading), as a little-endian four-bytes count of
seconds since **01/01/2000 @ 00:00**.

This differs from the otherwise similar UltraEasy protocol, that uses
the UNIX Epoch of **01/01/1970 @ 00:00**.
