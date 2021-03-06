<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# TaiDoc TD-42xx

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.com). With
thanks to Gianni Ceccarelli for the time fields structure.

The protocol is compatible with both TD-4277 and TD-4235B devices at least.

These devices are marketed as:

  * GlucoRx Nexus
  * GlucoRx NexusQ
  * Menarini GlucoMen Nexus
  * Aktivmed GlucoCheck XL

## Important device notes

The device has a miniUSB-A connector and comes with an onboard HID-to-Serial
adapter, based on the Silicon Labs cp2110 interface.

### USB IDs

| Device         | Vendor ID | Product ID |
| ---            | ---       | ---        |
| GlucoRx Nexus  | 10c4      | ea80       |
| GlucoRx NexusQ | 10c4      | ea80       |

### Serial port configuration

The CP2110 bridge should be configured as follows:

  * 8 data bits;
  * no parity bits;
  * 1 stop bit;
  * 19200 baud rate.

### Supported Features

The meter hardware supports:

  * Factory-fixed mg/dL or mmol/L *user* reported unit. Values are always
    returned by the protocol in mg/dL.
  * Meal flags (none, pre-meal, post-meal).
  * Sound and reminder alarms.

## Protocol

### Packet Structure

The device uses a simple challenge and response serial protocol with fixed-size
8-bytes commands and responses.

    packet = STX command-id message direction checksum
    STX = %x51

    command-id = OCTET
    message = 4OCTET
    empty-message = %x00 %x00 %x00 %x00

    direction = direction-in / direction-out
    direction-in = %xA5
    direction-out = %xA3

    checksum = OCTET

The checksum is calculated as a full sum of the first seven bytes of the packet,
and then truncated to 8-bit.

### Connection

Connection is established by sending a connection command, and receiving a
response.

    message-connect = %x00
    command-connect = %x22
    response-connect = %x22 / %x24 / %x54

In both the request and long  response packets, the message is zero.

### Get Model

To confirm the model of the device (probably to avoid confusing with other
devices sharing a similar protocol), you can send command `0x24`.

    model-request = STX %x24 empty-message direction-out checksum
    model-response = STX %x24 model OCTET OCTET direction-in checksum

    model = OCTET %x42  ; 16-bit little-endian value

The model will be reported as a 16-bit little-endian BCD value.

### Date and Time

#### Format

All representation of time for this protocol are idential, and they follow this
format:

    datetime = day minute hour
    day = 2OCTET  ; 16-bit little endian value.
    minute = OCTET
    hour = OCTET

While minutes and hours are left unpacked, the `day` field packs the full year,
month and day into a single 16-bit integer.

Within the 16-bit `day` field, the first 7 bits represent the year, starting
with 2000, followed by 4 bits for the month (range 1-12), and 5 bits for the day
(range 1-31).

#### Getting Time

To retrieve the time of the device, you send the command `0x23` with a zero
message, and parse the returned message as the date and time.

    gettime-request = STX %x23 empty-message direction-out checksum
    gettime-response = STX %x23 datetime direction-in checksum

#### Setting Time

To set the time of the device, you send the command `0x33` with the time as
message.

    settime-request = STX %x33 datetime direction-out checksum
    gettime-response = STX %x33 datetime direction-in checksum

### Reading Records

To retrieve the records from the device, you need to first request the number of
records, and then retrieve the timestamp and the value and flags for each
record.

Record count and record index are 16-bit unsigned integers, little-endian, and
are 0-indexed, with the most recent result first.

    record-id = 2OCTET  ; 16-bit little-endian value

#### Record Count

To retrieve the record count, send command `0x2B`.

    recordcount-request = STX %x2B empty-message direction-out checksum
    recordcount-response STX %x2B record-count unknown-value direction-in checksum

    record-count = 2OCTET  ; 16-bit little-endian value
    unknown-value = 2OCTET  ; likely 16-bit little-endian value

The `unknown-value` is unknown but appears to be bound to the record-count. For
a new meter with a record count of 0 (or a cleared meter), the unknown value is
always `0xFFFF`.

#### Record Timestamp

To retrieve each record timestamp, send command `0x25`.

    record-timestamp-request = STX %x25
                               record-id %x00 %x00 direction-out checksum
    record-timestamp-response = STX %x25 datetime direction-in checksum

#### Record Value

To retrieve each record value, send command `0x26`.

    record-value-request = STX %x26 record-id %x00 %x00 direction-out checksum
    record-value-response = STX %x26
                            glucose-value unknown meal-flag direction-in checksum

    glucose-value = 2OCTET  ; 16-bit little-endian value
    unknown = OCTET
    meal-flag = meal-none / meal-before / meal-after
    meal-none = %x00
    meal-before = %x40
    meal-after = %x80

### Clearing Memory

To completely clear the memory of the device, send command `0x52`.

    clear-memory = STX %x52 empty-message direction-out checksum
