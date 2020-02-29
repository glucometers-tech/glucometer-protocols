# FreeStyle Libre 2

Reverse engineering in progress.


## Important device notes

### USB IDs

| Device            | Vendor ID | Product ID |
| ---               | ---       | ---        |
| FreeStyle Libre 2 | 1a61      | 3950       |

## Protocol

This device uses the [shared HID protocol](shared-hid-protocol.md) used by other
meters in the FreeStyle family, but introduces encryption.

Text commands are sent by the original software as message type `0x21`, with
responses as `0x60`.

## Encryption

The commands sent to Libre 2 devices are encrypted. The encryption covers the 63
bytes remaining following the message type, for most of the message types used
by the device, excluding the pre-initialisation commands (and their replies),
the `0x22` keep-alive command, and the error replies.

The initialization appears to follow a handshake as follows:

<seqdiag>
{
    edge_length = 300;
    span_height = 20;
    default_fontsize = 12;

    software => reader [label = "0x05 QUERY SERIAL", return =  "0x06 SERIAL NUMBER"]

    software -> reader [label = "0x14, 0x11 REQUEST CHALLENGE"]
    software <-- reader [label = "0x33, 0x16 CHALLENGE"]

    software -> reader [label = "0x14, 0x17 CHALLENGE RESPONSE"]
    software <-- reader [label = "0x14, 0x18 UNKNOWN"]

    software -> reader [label = "0x01 INIT"]
    software <-- reader [label = "0x071 INIT COMPLETE"]
}
</seqdiag>
