<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# OneTouch Verio IQ

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.com), based
on
[Tidepool](https://github.com/tidepool-org/chrome-uploader/blob/master/lib/drivers/onetouch/oneTouchVerioIQ.js).

The communication protocol as described in this document applies to the
following models:

 * *OneTouch Verio IQ*

## Important device notes

The device has a miniUSB-A connector, and comes with an onboard USB-to-Serial
adapter compatible with the cp210x driver.

### USB IDs

| Device             | Vendor ID | Product ID |
| ---                | ---       | ---        |
| OneTouch Verio IQ  | 10c4      | 85a7       |

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* no parity bits;
* 1 stop bit;
* 38400 baud rate.

## Packet Structure

The device uses the same [Shared Binary Protocol](shared-binary-protocol.md) as
other LifeScan devices.

Within the packet, the `link-control` byte is unused and always left at
`0x00`.

The `command-prefix` byte may have one of accepted values as defined below, but
for ease of implementation, `0x03` is suggested. This matches
the [OneTouch Verio 2015](onetouch-verio-2015.md).

    packet                  ; see shared-binary-protocol.md
    link-control = %x00     ; not used by this device
    command-prefix = %x03 / ; suggested
                     %x04
## Timestamp Format

Timestamps, both for the device's clock and for the reading records, are defined
as a little-endian 32-bit number, representing the number of seconds since
**2000-01-01 00:00:00**.

It should not be mistaken for a UNIX timestamp, although the format is
compatible. To convert to UNIX timestamp, you should add `946684800` to the
value (the UNIX timestamp of the device's own epoch.)

## Messages

Messages are binary, and only some are related to each other in any obvious way.

The commands have been named after their function, in the style of SCSI commands
(as done for the [Verio 2015](onetouch-verio-2015.md):

 * **READ VERSION** to read the firmware version of the device.
 * **READ SERIAL** to read the device serial number.
 * **READ RTC** to retrieve current RTC time of the device.
 * **WRITE RTC** to change the RTC time fo the device.
 * **READ UNIT** to retrieve the device blood sugar display unit.
 * **READ RECORD COUNT** to retrieve the number of records in the device's
   memory.
 * **READ RECORD** to retrieve the content of one record.
 * **ERASE MEMORY** to clear the meter altogether.

### READ VERSION

    READ-VERSION-request = STX %x09 %x00 ; message length = 9 bytes
                           %x03 %x0d %x01 ETX checksum

    READ-VERSION-response = STX length %x00
                            %x03 %x06 version-length
                            [version-length]VCHAR %x00
                            ETX checksum

The version string is encoded in ASCII and prefixed with its length but also
appears NULL-terminated. This may be incorrect, and it may be that the string is
only prefixed, and padded to 18 characters.

### READ SERIAL

    READ-SERIAL-request = STX %x0a %x00 ; message length = 10 bytes
                          %x03 %x0b %x01 %x02 ETX checksum

    READ-SERIAL-response = STX length %x00
                           %x03 %x06 serial-number
                           ETX checksum
    serial-number = *VCHAR %x00

The serial number is encoded in ASCII and NULL-terminated.

### READ RTC

    READ-RTC-request = STX %x09 %x00 ; message length = 9 bytes
                       %x03 %x20 %x02
                       ETX checksum

    READ-RTC-response = STX %x0c %x00 ; message length = 12 bytes
                        %x03 %x06 timestamp
                        ETX checksum

    timestamp = 4OCTET ; 32-bit little-endian value

### WRITE RTC

    WRITE-RTC-request = STX %x0d %x00 ; message length = 13 bytes
                        %x03 %x20 %x01 timestamp
                        ETX checksum

    WRITE-RTC-response = STX %x08 %x00 ; message length = 8 bytes
                         %x03 %x06
                         ETX checksum

### READ UNIT

    READ-UNIT-request = STX %0a %x00 ; message length = 10 bytes
                        %x03 %09 %02 %02
                        ETX checksum

    READ-UNIT-response = STX %x0c %x00 ; message length = 12 bytes
                         %x03 parameter-unit-mgdl / parameter-unit-mmoll
                         %x00 %x00 %x00
                         ETX checksum

    parameter-unit-mgdl = %x00
    parameter-unit-mmoll = %x01

### READ RECORD COUNT

    READ-RECORD-COUNT-request = STX %x09 %x00 ; message length = 9 bytes
                                %x03 %x27 %x00
                                ETX checksum

    READ-RECORD-COUNT-response = STX %x0a %x00 ; message length = 10 bytes
                                 %x03 %x06 message-count
                                 ETX checksum
    message-count = 2OCTET ; 16-bit little-endian value

### READ RECORD

The records are then accessed through indexes between 0 and `message-count`
(excluded) as reported by **READ RECORD COUNT**.

    READ-RECORD-request = STX %x0a %x00 ; message length = 10 bytes
                          %x03 %x21 record-number
                          ETX checksum
    record-number = 2OCTET ; 16-bit little-endian value

The record number is assumed to be a 16-bit little endian value, for consistency
with other LifeScan devices.

    READ-RECORD-response = STX %x12 %x00 ; message length = 18 bytes
                           %x03 %x06 timestamp glucose-value
                           control-flag flag-meal
                           %x00 %x00
                           ETX checksum

    glucose-value = 2OCTET         ; 16-bit little-endian value

    control-flag = not-control / control
    not-control = %x00
    control = %x01

    flag-meal = meal-none / meal-before / meal-after
    meal-none = %x00
    meal-before = %x01
    meal-after = %x02

The `glucose-value` is represented as a 16-bit little endian value. It represent
the blood sugar in mg/dL. As most other meters, the eventual conversion to
mmol/L happens only at display time.

The `control-flag` is a byte representation of a boolean. If true, the record
refers to a control solution test, rather than a blood measurement.

The `meal-flag` is a single byte, representing a tristate of no information,
before meal reading, and after meal reading. This meal information cannot be set
on the Verio meter, but can be set on the Select Plus.

### MEMORY ERASE

The memory erase command deletes all the records in the device's
memory.

    MEMORY-ERASE-request = STX %x08 %x00 ; message length = 8 bytes
                           %x03 %x1a
                           ETX checksum

    MEMORY-ERASE-response = STX %x08 %x00 ; message length = 8 bytes
                            %x03 %x06
                            ETX checksum

Remember that this action is irreversible.
