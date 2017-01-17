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

The `message-type` is not fully understood, and its values differ
between different devices, and are thus not part of this document.

The `message-length` represent the number of significant bytes in the message
*excluding type and length*.

## Initialization Sequence

The devices following this protocol share the same initialization sequence,
sending four messages with no payload. The content of the replies could be used
to validate whether the device is the one expected, but their meaning is not
fully understood.

    init-command = %x04 %x00
    init-command-reply = %x34 %x01 OCTET

The octet returned by this command appears not to be constant per device,
according to Xavier's notes and experimentation. It is not constant across
devices. No meaning is known for this.

    init-query-serial = %x05 %x00
    init-query-serial-reply = %x06 %x0e
                              ( serial-number / no-serial-number ) %x00
    serial-number = 7( ALPHA / DIGIT ) "-" 5( ALPHA / DIGIT )
    no-serial-number = "00000000 (No SerialNum)"

Note: the serial number definition is common to most Abbott device, except newer
devices have a longer, and non-compatible serial number format. It is possible
that the Abbott software can determine the driver to use based on this serial
number.

    init-query-swver = %x15 %x00
    init-query-swver-reply = %x35 message-length software-version %x00
    software-version = VCHAR *( VCHAR / SP )

The software version definition is effectively free-form. Newer models appear to
include a date.

    init-complete = %x01 %x00
    init-complete-reply = %x71 %x01 %x01

This is the same exact request/reply pair in all the currently observed devices.

## Text commands

Most of the devices using this HID-based protocol can be sent commands that
receive ASCII text responses. These responses can span multiple frames, allowing
the single response to cross the 62-bytes length limit.

Each text response, once reconstructed, complies to the following specs:

    response = message "CKSM:" checksum CRLF
               "CMD" SP command-status CRLF

    checksum = 8HEXDIG               ; all "0" on failure
    message = *( VCHAR / SP / CRLF ) ; only present on success
    command-status ("OK" / "Fail!")

The `<checksum>` is calculated by summing up the ASCII value of each of the
bytes forming the message, including the final newline that usually terminates
the message.

Since no message is present in case of message failure, the checksum is null and
is represented by the string `00000000`.

## Multiple records commands

Different commands can return a list of records. While their meaning depends on
the issued command, the reported format fits the same syntax:

    multi-records = *( record CRLF )
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
    month = 1*2DIGIT
    day = 1*2DIGIT
    year 1*2DIGIT

Fetch or set the current date as seen by the device. If a date value is passed
following the command, it'll be interpreted as a set-date.

Note the year value is defined at most with two digits, the value 2000 needs to
be added (or subtracted) to match the current year.

In case of successful setting of date, the command is confirmed with no further
information; in case of error, the `<message>` field will report the encountered
error.

### `$time`

    time-cmd = "$time" ("?" / "," time-value)
    time-query-response = time-value CRLF

    time-value = hour "," minute
    hour = 1*2DIGIT
    minute = 1*2DIGIT

Fetch or set the current time as seen by the device.

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
variable is not accepted.

### `$ptid`

    ptid-cmd = "$ptname" ("?" / "," patient-id)
    ptid-query-response = patient-id CRLF

    patient-id-setting = 1*VCHAR
    patient-id-response = *VCHAR

Returns the patient ID as configured in the device.

Note that while an empty response is valid, an empty value when setting the
variable is not accepted.
