# Abbott Shared HID protocol

Abbott devices such as FreeStyle InsuLinx and FreeStyle Libre share a
common communication protocol based on USB HID. The commands executed
on top of this protocol are not cross-device compatible.

Credit for the reverse engineering go to Xavier Claessens for
[OpenGlucose](https://github.com/xclaesse/OpenGlucose/blob/master/src/insulinx.c).

## Communication Protocol

These devices appears on the USB bus as a HID device. The
communication between the software and the device happens through HID
**Set Report**/**Get Report** interfaces, in a way that is compatible
with the
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

The device follows the same initialization protocol as the InsuLinx, sending
four messages with no payload, although not all the replies are currently
meaningful.

    init-command = %x04 %x00
    init-command-reply = %x34 %x01 OCTET

The octet returned by this command appears not to be constant per device,
according to Xavier's notes, and definitely is not constant across devices. No
meaning is known for this.

    init-query-serial = %x05 %x00
    init-query-serial-reply = %x06 %x0e serial-number %x00
    serial-number = 7( ALPHA / DIGIT ) "-" 5( ALPHA / DIGIT )

Note: the serial number definition is common to all Abbott devices. It is
possible that the serial number itself identifies the device model.

    init-query-swver = %x15 %x00
    init-query-swver-reply = %x35 message-length software-version %x00
    software-version = DIGIT "." DIGIT "." DIGIT /  ; on my Libre
	                   DIGIT "." 3DIGIT             ; on InsuLinx

The software version definition is different between models, it appears.

    init-complete = %x01 %x00
    init-complete-reply = %x71 %x01 %x01

This is the same exact request/reply pair in both Libre and InsuLinx.
