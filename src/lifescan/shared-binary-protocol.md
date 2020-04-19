<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# LifeScan Shared Binary Protocol

Multiple LifeScan devices of the OneTouch family share a common packet framing
protocol, clearly designed for serial communication but used over different
transmission protocols as well.

While commands within the protocol are specific to the model, the framing is
vastly similar if not identical.

    packet = STX length link-control command_prefix message ETX checksum
    STX = %x02
    length = OCTET
    link-control = OCTET
    message = [length - 6]OCTET
    ETX = %x03
    checksum = 2OCTET           ; 16-bit little-endian value

The packets are variable length, with the length provided by the byte following
the `STX` constant. This length is inclusive of the whole packet framing. The
message itself is 6 bytes short of this value.

The `STX` and `ETX` constants mark start and end of the message and correspond
to the ASCII characters bearing the same names.

The `link-control` byte is only used by some devices, and for others is
maintained to a constant `0x00`.

The `checksum` is a variant of CRC-16-CCITT, seeded at `0xFFFF`, and stored
little-endian.

## Link Control

The `link-control` byte is known used only in
the [OneTouch Ultra Easy](onetouch-ultraeasy.md) protocol, which contains a
description of the field. As no other meter uses this byte, its documentation is
left to the LifeScan provided specifications.

## Command Prefix

For packet that includes non-zero length messages, the message itself can be
defined as

    message = command-prefix *OCTET
    command-prefix = OCTET

The command prefix is device-specific, and is present both in the requests sent
and the responses received.

Some devices allow more than one prefix to be used, although not in the same
session. If multiple prefixes are legal, the responses will echo back the
command prefix of their request.

Exceptions (likely bugs) exists where this guarantee is not maintained, check
device-specific documentation.

## Command Success

With the exception of a few device-specific control packets, all commands sent
to the device receive a response in the same packet format. This extends the
format a response to:

    response = command-prefix status *OCTET
    command-prefix = OCTET ; as provided in the request
    status = success / error
    success = %x06
    error = %x09 / OCTET ; only some errors are identified

Whether data follows a non-success `status` is device specific. Data should not
be considered a valid response if the `status` is not `success`.
