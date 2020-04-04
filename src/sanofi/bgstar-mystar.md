# Sanofi BGStar and Mystar Extra

Reverse Engineered by Noury (@nbenm)

## Important device notes

This protocol is used in the following devices:

 * BGStar
 * MyStar Extra
 * Other devices exist but have not been tested

## Cable

These devices connect to a computer with a USB to RS232 cable.

Sanofi provides for free this cable.
It contains a CP210x component, and uses a VCP CP210x driver.
This cable is known as Zero-click cable.

With linux cp210x and usbserial modules are loaded.
A device is created in /dev/ttyUSBx
With MacOs it uses Mac VCP Driver (CP210x) from Silicon Labs.
a device is created in /dev/cu.SLAB_USBtoUART
I have asked them in 2016 to add the vendor id/product id, and they did it.

## Protocol

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* no parity bits;
* 1 stop bit;
* 115200 baud rate.

### Messages and commands

All commands (requests) must end with a carriage return.

In all results, leading zeros are removed.

Message status is reported with HTTP-style status codes.

    continue = "100"
    ok = "200"

## Hello command

    first-command = "hello" CR

    first-response = ok SP "hello" SP name
    name = 4ALPHA "-" 2ALPHA

## Get Serial command

    get-serial-cmd = "get serial" CR
    get-serial-response = ok SP "serial" SP serial
    serial = 14( ALPHA / DIGIT )

## Get date and time

    get-datetime-cmd = "get datetime" CR
    get-datetime-response = ok SP datetime

    datetime = year SP month SP day SP
               hour SP minutes SP seconds
    year = 4DIGIT
    day = 1*2DIGIT
    hour = 1*2DIGIT
    minute = 1*2DIGIT
    second = 1*2DIGIT

## Set date and time

TODO: confirm the correct command.

## Get number of results

    get-glucount-cmd = "get glucount" CR
    get-glucount-response = ok SP "glucount" 1*4DIGIT

## Get single result

Result ID is in reverse time order, 0 is most recent.

    get-glurec-cmd = "get glurec" SP glurec-id CR
    glurec-id = 1*4DIGIT

    get-glurec-response = ok SP "glurec" SP record
    record = 1DIGIT 1DIGIT value flag-meal datetime
    value = 1*3DIGIT

    flag-meal = no-meal /
                before-breakfast / after-breakfast /
                before-lunch / after-lunch /
                before-dinner / after-dinner
    no-meal = "0"
    before-breakfast = "1"
    after-breakfast = "2"
    before-lunch = "3"
    after-lunch = "4"
    before-dinner = "5"
    after-dinner = "6"

If there was an error when the measure was taken, response begins with E.
Unfortunately, I can't reproduce this to have the full error message.  But it is
sure that fifth field begins with an E (fields are separated by spaces)

In the meter I have used, value is in mg/dL

The maximum number of results is 1865.
After that oldest result is replaced.

To be checked: after that which number is used for a new result?

##  Get System Info

TODO: confirm whether lines end with CR or CRLF.

    get-sysinfo-cmd = "get sysinfo all" CR
    get-sysinfo-response = 1*sysinfo-line
                           ok SP "sysinfo all"
    sysinfo-line = continue SP sysinfo-key SP sysinfo-value CR
    sysinfo-key = "model" / "product" / "comm" / "commaster" /
                  "co" / "firmware" / "calcode" / "compiler" /
                  "compiletime" / "deviceid" / "cpu" / 1*ALPHA
    sysinfo-value = 1*VCHAR

## Get Glucose Units:

    get-glucose-unit-cmd = "get gluunit" CR
    get-glucose-unit-response = ok SP "gluunit" SP unit
    unit = "mg/dL" / 1*VCHAR

TODO: confirm the unit for mmol/L devices.
