# SD Codefree

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.eu).

## Cable

The data cable that you can buy together with the device is an USB Serial
adapter, based on Silicon Laboratories CP210x chipset.

It appears to be a fairly standard USB-to-TRS serial adapter, though the
connector is a subminiature (2.5mm connector.)

## Protocol

The communiation happens over a serial binary protocol. The device initiates the
connection once turned on, and the computer needs to respond to the opening
challenge and follow up packets from the device with either commands or
acknowledgments.

If the device receives an invalid command, it reports `E-5` on the display and
needs to be turned off.

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* no parity bits;
* 1 stop bit;
* 38400 baud rate.

### Message structure

Messages follow a simple format:

    message = STX direction length message checksum ETX
    STX = %x53
    direction = direction-in / direction-out
    direction-in = %x20
    direction-out = %x10
    length = OCTET
    message = [length-5]OCTET
    checksum = OCTET
    ETX = %xAA

The messages are variable length, with the following length provided in the
third byte of the message (length does not include the three bytes up to that
point). The second byte is a direction indication (device-to-host and
host-to-device).

The second to last byte is a checksum calculated as an 8-bit bitwise xor of the
`message` bytes.

### Protocol initialization

When the device is turned on it sends the following packet on the wire:

    challenge = STX direction-in %x04 %x10 %x30 checksum ETX

Sometimes a leading null byte might be read from the message, unclear if this is
due to the USB adapter, a bug in the driver or the device.

If the device is not given a response within time, it ignores the cable. The
expected response is:

    response = STX direction-out %x04 %x10 %x40 checksum ETX

Following the response, the device will send a packet that is composed mostly of
`0xAA` values, which include the count of readings in the device:

    count-packet = STX direction-in %x18 %x30 readings-count [19]%xAA checksum ETX

    readings-count = 2 OCTET ; 16-bit big-endian value

The count of reading is assumed to be 16-bit because the manual references a
memory of 1000 readings.

### Setting date and time

The only parameter that can be tweaked on the device appears to be the date and
time.

This is done through what appears like a text command sent over the binary
protocol.

The packet need to be sent right after initialization and is acknowledge by the
meter.

    set-time-packet = STX direction-out %x13
                      'ADATE' set-year set-month set-day
                      set-hour set-minute checksum ETX
    set-year = 4DIGIT
    set-month = 2DIGIT
    set-day = 2DIGIT
    set-hour = 2DIGIT
    set-minute = 2DIGIT

    set-time-ack-packet = STX direction-in %x04 %x10 %x10 checksum ETX

Note that while the date is set with a full four-digits year, the device will
ignore the first two digits — 1999, 2099 and 2199 appear as exactly the same
value internally.

After receiving the acknwoeldgement, the device need to be told to disconnect,
as otherwise it'll be stuck in PC connection mode. This can be done with the
same packet used to fetch readings (see later in this document).

    disconnect-packet = STX direction-out %x04 %x10 %x60 checksum ETX
    disconnect-ack-packet = STX direction-in %x04 %x10 %x70 checksum ETX

### Dumping the readings history

Alternatively from setting the date, it is possible to dump the records from the
device. The readings will be returned in inverse order of them being taken, one
for each fetch packet:

    fetch-packet = STX direction-out %x04 %x10 %x60 checksum ETX
    reading-packet = STX direction-in %x20 %x13 OCTET year month day
                     hour minute value flag-meal 7OCTET checksum ETX
    year = OCTET
    month = OCTET
    day = OCTET
    hour = OCTET
    minute = OCTET
    value = 2OCTET ; 16-bit big-endian value

    flag-meal = no-meal / before-meal / after-meal
    no-meal = %x00
    before-meal = %x10
    after-meal = %x20

The last seven bytes of the message are not understood yet, they seem to change
independently from any settings on the device or on the reading itself, but they
also don't change enough to look like a checksum of any kind.

After the last reading is provided, a further fetch causes a disconnection.
