# FreeStyle Freedom Lite / Lite / Mini

Reverse Engineered by [Alexander
Schrijver](http://www.flupzor.nl/protocol.html) (@flupzor).

## Important device notes

This protocol is used in the following devices:

 * FreeStyle Freedom Lite
 * FreeStyle Lite
 * FreeStyle Mini

## Cable

These devices connect to a computer with a TRS (mini-jack) to RS232 cable.

Abbott provides a TIUSB3410 USB-to-Serial adapter by Texas Instruments. This
cable is supported at least from Linux 3.14 and later.

## Protocol

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* no parity bits;
* 1 stop bit;
* 19200 baud rate.

### Messages and commands

The FreeStyle protocol has 2 types of messages a command and a result. Currently
only one command exists which returns multiple results.

    generic-message = command / result

The host starts with setting up the connection. After that it should send the
following command:

    command = "mem"

The device then sends the following data.

    result = CR LF
             serial-number CR LF
             software-version CR LF
             currentdatetime
             log

    serial-number = 7( ALPHA / DIGIT ) "-" 5( ALPHA / DIGIT )
    software-version = CR LF 1*DIGIT "." 1*DIGIT *SP "-P"

The serial number is provided in the same format as other FreeStyle devices.

The software revision doesn't seem to be very consistent. It usually includes a
dot-version, as a normal software revision, and terminate with the `-P`
string. Sometimes the version is space-padded, and sometimes it's not.

### Date and Time Format

    currentdatetime = datetime

    datetime = month SP day SP year SP time
    month = "Jan " / "Feb " / "Mar " / "Apr " / "May " / "June" / "July" /
        "Aug " / "Sep " / "Oct " / "Nov " / "Dec "
        ; N.B. Except June and July all months have spaces at the end.
    day = 2DIGIT
    year = 4DIGIT

    time = hour ':' minute ':' second
    hour = 2DIGIT
    minute = 2DIGIT
    second = 2DIGIT

### Log

    log = empty-log / log-contents
    empty-log = "Log Empty END"
    log-contents = CR LF
                   nrresult CR LF LF
                   resultline *(CR LF resultline)
                   checksum "END"

    nrresults = 3DIGIT; e.g. 001 or 250

    resultline = result-value datetime2 plasmatype "0x00"
    result-value = 3DIGIT
    datetime2 = month SP day SP year SP time2
    time2 = hour ':' minute
    plasmatype = 2DIGIT

    checksum = "0x" 4HEXDIG

There is probably is a maximum number of results. At the moment we are guessing
it is 450 which is the maximum the Abbott Precision Xceed can store. The minimum
is 1

Note that the inconsistent spacing (including the additional LF) is intentional.

NOTE: `result-value` minimum and maximum are not known; comparing with other
Abbott device, it's possible that they use the constants `HI` and `LO` instead.

The `checksum` is a simple accumulation of the ASCII values of the characters,
up until the `0x` literal (excluded), including the `CR LF` terminations.
