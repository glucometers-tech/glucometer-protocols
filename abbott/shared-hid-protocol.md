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
