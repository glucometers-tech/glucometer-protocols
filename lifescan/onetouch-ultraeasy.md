# OneTouch Ultra Easy

The OneTouch Ultra Easy protocol specifications have been made available by
LifeScan in the past, and as such require no reverse engineered protocol
specification.

This document is intended to only note similarities with other OneTouch devices.

The communication protocol applies to the following models:

 * *OneTouch Ultra Easy*
 * *OneTouch Ultra Mini*

## Communication

The device communicates over a TRS-connected serial port, compatible with
the [OneTouch Ultra2](onetouch-ultra2.md) device.

The packet structure of the protocol follows
the [Shared Binary Protocol](shared-binary-protocol.md).

Within the packet, the `link-control` byte is used; refer to the LifeScan
provided specifications.

The `command-prefix` byte is expected to be `0x05`.

    packet                  ; see shared-binary-protocol.md
    link-control = OCTET    ; used by this device
    command-prefix = %x05
