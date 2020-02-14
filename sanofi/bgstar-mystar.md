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

## Hello command

First command to send is hello\r  
Response is 200 hello name => name = 4( ALPHA ) "-" 2( ALPHA )  
Example: 200 hello JAZZESC-EN

## Get Serial command

**request:** get serial\r  
**response:** 200 serial 14( ALPHA / DIGIT )  
Example: 200 serial JBAA211G300702  

## Get datetime
**request:** get datetime\r  
**response:** 200 datetime year month day hour minutes seconds  
Example: 200 datetime 2020 2 14 21 30 2  
  
year is on 4 digits  
month is on 1 or 2 digits  
day is on 1 or 2 digits  
hour is on 1 or 2 digits  
minutes is on 1 or 2 digits  
seconds is on 1 or 2 digits  
  
## Set datetime  
**request:** set datetime\r  
**response:** 200 datetime year month day hour minutes seconds  
Example: 200 datetime 2020 2 14 21 30 2  
  
year is on 4 digits  
month is on 1 or 2 digits  
day is on 1 or 2 digits  
hour is on 1 or 2 digits  
minutes is on 1 or 2 digits  
seconds is on 1 or 2 digits  
  
## Get number of results  
**request:** get glucount\r  
**response:** 200 glucount 1 to 4 digits  
Example: 200 glucount 935  
  
## Get one result by position (last is zero)  
**request:** get glurec number\r => number is 1 to 4 digits  
**response:** 200 glurec unknown unknown value type date_and_time  
Example: 200 glurec 0 0 113 1 2020 2 13 8 34 18  
  
Type:  
  
1:Before Breakfast  
2:After Breakfast  
3:Before Lunch  
4:After Lunch  
5:Before Dinner  
6:After Dinner  
0:Other  
  
date_and_time: is same format as in get datetime  
  
value: 1 to 3 digits  
  
If there was an error when the measure was taken, response begins with E.  
Unfortunately, I can't reproduce this to have the full error message.  
But it is sure that fifth field begins with an E (fields are separated by spaces)  
  
In the meter I have used, value is in mg/dL  
  
##  Get System Info:  
  
I don't have detail. It's not necessary.  
Here's an example:  
  
**request:** get sysinfo all\r  
**response:**  
100 model JAZZESC-EN  
100 product BGStar  
100 comm F  
100 commmaster F  
100 co A  
100 firmware 4.8.11.b1.34  
100 calcode 840010042A  
100 compiler MSP430 IAR C/C++ version 5.10.4  
100 compiletime Nov 19 2010 16:25:43  
100 deviceid 0xf449  
100 cpu MSP430F449  
200 sysinfo all  
  
## Get Glucose Units:  
  
**request:** get gluunit\r  
**response:** 200 gluunit mg/dL  
  
The maximum number of results is 1865.  
After that oldest result is replaced.  
  
To be checked: after that which number is used for a new result?
