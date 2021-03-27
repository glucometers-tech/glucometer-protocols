<!--
SPDX-FileCopyrightText: 2021 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# GlucoMen Areo

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.com).

## Cable

The data cable provided for the device is an USB Serial adapter based on Silicon Laboratories CP210x chipset.

It appears to be a fairly standard USB-to-TRS serial adapter, though the
connector is a subminiature (2.5mm connector):

| Connector | Meaning        |
| ---       | ---            |
| Tip       | Host-to-Device |
| Ring      | Device-to-Host |
| Sleeve    | GND            |

### USB IDs

| Device              | Vendor ID | Product ID |
| ---                 | ---       | ---        |
| GlucoMen Areo Cable | 10c4      | ea60       |

## Protocol

The communication happens over a serial protocol, mixing text-based and byte-based commands.

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* odd parity;
* 1 stop bit;
* 9600 baud rate.

### Request and Response Structure

Commands are sent from the host to the device with either a single byte, or with two command bytes
followed by text parameters. Responses from the device are either in text format, or a single byte
response (if the request had text parameters).

There are very few known commands that can be represented as such:

    request = get-info-command / get-readings-command / set-datetime-command
    response = text-parameters / no-readings / simple-response

    get-info-command = %xA2
    get-readings-command = %x80
    set-datetime-command = %xC2 %xA1 text-parameters

    simple-response = F / P
    no-readings = "[" CRLF %x90 %x3D CRLF "]" CRLF
    text-parameters = "[" CRLF *response-line checksum CRLF "]" CRLF

    response-line = 1*VCHAR CRLF
    checksum = 2HEXDIG

### Checksum

The checksum is calculated according to the CRC-8/Maxim algorithm, applied on the text parameters
starting from the opening square bracket (`[`) up until and including the `CRLF` preceding the checksum itself.

It is represented as two **upper case** hexadecimal digits. The device will refuse a command sent
with a lower-case checksum even if correct.

### `GET INFO`

The `GET INFO` command receives a text response consisting of a single line, comma-separated:

    get-info-response = *DIGIT "," *DIGIT "," *DIGIT "," serial-number "," sw-version CRLF
    serial-number = *SP 1*VCHAR
    sw-version = *SP 1*VCHAR

The first three values are unknown at the time of writing, while the serial number and software
versions may have leading spaces, and are otherwise printable characters.

### Setting Date time

To set the date and time of the device, a single command can be sent, followed by the text
parameters in the same format as returned by the device.

Note: the checksum has to be in upper case to be accepted by the device.

    set-datetime-command = %xC2 %xA1 "[" CRLF datetime checksum CRLF "]" CRLF
    datetime = year month day hour minute
    year = 2DIGIT
    month = 2DIGIT
    day = 2DIGIT
    hour = 2DIGIT
    minute = 2DIGIT

### Readings

The `GET READING` command receives either a response informing that there are no readings on the
device or one reading per line in the text format with comma-separated values.

    no-readings = "[" CRLF %x90 %x3D CRLF "]" CRLF

    reading-response-line = reading-type "," value "," unit "," markings "," date "," time CRLF
    reading-type = "Glu" / 1*VCHAR
    value = 1*DIGIT *1("." DIGIT)
    unit = "mmol/L" / 1*VCHAR
    markings = no-marking / check-mark / before-meal / after-meal / exercise
    date = year month day
    time = hour minute

    no-marking = "00"
    check-mark = "01"
    before-meal = "02"
    after-meal = "04"
    exercise = "08"

    year = 2DIGIT
    month = 2DIGIT
    day = 2DIGIT
    hour = 2DIGIT
    minute = 2DIGIT

It's important to note that the readings are expressed in the native unite of the device and not,
like in most other meters, in mg/dL even for mmol/L devices. This means you need to check the
unit that is used in the same reading itself.

The markings values appear to correspond to values in a bitmask but they cannot be applied
simultaneously, so they should be considered as different enumeration values.
