<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Abbott Shared HID protocol

Multiple Abbott devices of the FreeStyle family share a common communication
protocol based on USB HID. These devices all provide a direct USB connector and
require no special cable nor driver to speak with a PC.

While most of the commands are not compatible between each other, the devices do
share a basic framing protocol and basic text command format.

Credit for the reverse engineering go to Xavier Claessens for
[OpenGlucose](https://github.com/xclaesse/OpenGlucose/blob/master/src/insulinx.c).

## Communication Protocol

These devices appears on the USB bus as a HID device. The communication between
the software and the device happens through HID **Set Report**/**Get Report**
interfaces, in a way that is compatible with the
[Linux hidraw](https://www.kernel.org/doc/Documentation/hid/hidraw.txt)
interface.

In particular the packets are sent on the *Control* endpoint, as *Class
Interface* messages (`bRequestType = 0x21, bRequest = 0x09, wValue = 0x0200,
wIndex = 0x0000`), and received on the interrupt endpoint (`0x81`). Report
numbers are not used.

The "reports" are not following the HID standard. They are of a fixed 64-bytes
size, even if most of the commands themselves are shorter. Replies can span
multiple inbound reports.

    message = message-type message-length 62OCTET
    message-type = OCTET  ; this will more properly defined later.
    message-length = %x00-3E

The `message-type` is not fully understood, and its valid values differ
between different devices, so please reference the specific device document.

The `message-length` represent the number of significant bytes in the message
*excluding type and length*.

Some devices (notably the Libre 2 reader) encrypt the message following
`message-type`, and as such don't have a valid `message-length` value.

## Binary Messages

Abbott's original software sends a number of pre-initialization commands to
identify the model and software version of the device, and choose the
appropriate protocol implementation. These are all binary (i.e. non-text)
commands. The names are arbitrary to provide a mnemonic reference.

### INIT (`0x01`)

    init = %x01 %x00
    init-reply = %x71 %x01 %x01

This is the same exact request/reply pair in all the currently observed
devices. Most devices will respond to text commands following this
initialization.

### `0x04`

    x04-command = %x04 %x00
    x04-command-reply = %x34 %x01 OCTET

The octet returned by this command appears not to be constant per device,
according to Xavier's notes and experimentation. It is not constant across
devices. No meaning is known for this.

### QUERY SERIAL (`0x05`)

    init-query-serial = %x05 %x00
    init-query-serial-reply = %x06 %x0e
                              ( serial-number / no-serial-number ) %x00
    serial-number = 7( ALPHA / DIGIT ) "-" 5( ALPHA / DIGIT )
    no-serial-number = "00000000 (No SerialNum)"

Note: the serial number definition is common to most Abbott device, except newer
devices have a longer, and non-compatible serial number format. It is possible
that the Abbott software can determine the driver to use based on this serial
number.

### QUERY SWVER (`0x15`)

    init-query-swver = %x15 %x00
    init-query-swver-reply = %x35 message-length software-version %x00
    software-version = VCHAR *( VCHAR / SP )

The software version definition is effectively free-form. Some newer models
appear to include a date.

## Error Conditions

### UNKNOWN COMMAND

Whenever an invalid (unknown) message type is sent to a device, it will respond
with the following message:

    error-reply = %x30 %x01 %x85

## Synchronization packets

Some devices send what appears like a "synchronization packet" every few reports
read, that is so defined:

    synchronization-packet = %x22 %x01 OCTET

The frequency of these packets appears to depend on the device (some devices
appear to send it exactly every three reports sent), and so does the content of
the packet.

## Text commands

Most of the devices using this HID-based protocol can be sent commands that
receive ASCII text responses. These responses can span multiple frames, allowing
the single response to cross the 62-bytes length limit.

Each text response, once reconstructed, complies to the following specs:

    response = message "CKSM:" checksum CRLF
               "CMD" SP command-status CRLF

    checksum = 8HEXDIG
    message = *( VCHAR / SP / CRLF )
    command-status ("OK" / "Fail!")

The `<checksum>` is calculated by summing up the ASCII value of each of the
bytes forming the message, including the final newline that usually terminates
the message.

Since no message is present in case of message failure, the checksum is null and
is represented by the string `00000000`.

## Multiple records commands

Different commands can return a list of records. While their meaning depends on
the issued command, the reported format fits the same syntax:

    multi-records = empty-log / log

    empty-log = "Log Empty" CRLF
    log = *( record CRLF )
          record-count "," checksum

    record = *( value "," ) value
    value = *VCHAR
    record-count = 1*DIGIT
    checksum = 8HEXDIGIT

The `<record-count>` field will contain the number of records that were
returned; the `<checksum>` field is calculated the same way as the overall
command one by summing up the ASCII value of each of the bytes forming the
record set, up to and including the final newline, and excluding the record
count.

## Common commands

There are a number of commands that appear shared across FreeStyle devices, and
are thus documented here to avoid duplication.

As devices may not support all these common commands, please refer to the
detailed protocol definition to note inconsistencies.

The majority of these commands are getters and setters of variables, some
free-form and some structured. The generic syntax for these commands is:

    command = "$" variable-name ( "?" / "," variable-value )
    variable-name = 1*VCHAR
    variable-value = *( VCHAR / SP )

Some variable (such as the software version and serial number) are
read-only. Those commands are documented with their full value, including the
terminating question mark.

### `$swver?`

    swver-response = 1* ( VCHAR / SP ) CRLF

Returns the software version of the device. The returned value is free-form, as
it changes among devices.

### `$serlnum?`

    serlnum-response = 1* ( ALPHA / DIGIT ) CRLF

Returns the serial number of the device. The serial number format appears to be
different from previous Abbott devices and as such is to be considered
free-form.

Please note that this command may provide a serial number even for devices that
respond with `<no-serial-num>` during initialization.

### `$date`

    date-cmd = "$date" ("?" / "," date-value)
    date-query-response = date-value CRLF

    date-value = ( month "," day "," year )
    month = 1*3DIGIT
    day = 1*3DIGIT
    year 1*3DIGIT

Fetch or set the current date as seen by the device. If a date value is passed
following the command, it'll be interpreted as a set-date.

If the device does not have a valid date settings (e.g. the real time clock lost
its power), the value 255 is reported for all fields.

Note the year value is defined at most with two digits, the value 2000 needs to
be added (or subtracted) to match the current year.

In case of successful setting of date, the command is confirmed with no further
information; in case of error, the `<message>` field will report the encountered
error.

### `$time`

    time-cmd = "$time" ("?" / "," time-value)
    time-query-response = time-value CRLF

    time-value = hour "," minute
    hour = 1*3DIGIT
    minute = 1*3DIGIT

Fetch or set the current time as seen by the device.

If the device does not have a valid time settings (e.g. the real time clock lost
its power), the value 255 is reported for all fields.

In case of successful setting of date, the command is confirmed with no further
information; in case of error, the `<message>` field will report the encountered
error.

### `$ptname`

    ptname-cmd = "$ptname" ("?" / "," patient-name)
    ptname-query-response = patient-name CRLF

    patient-name-setting = 1*VCHAR
    patient-name-response = *VCHAR

Returns the patient name as configured in the device.

Note that while an empty response is valid, an empty value when setting the
variable is not always accepted. The behaviour depends meter by meter.

### `$ptid`

    ptid-cmd = "$ptname" ("?" / "," patient-id)
    ptid-query-response = patient-id CRLF

    patient-id-setting = 1*VCHAR
    patient-id-response = *VCHAR

Returns the patient ID as configured in the device.

Note that while an empty response is valid, an empty value when setting the
variable is not accepted.
