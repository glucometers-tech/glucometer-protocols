# Glucometer protocols specifications

This repository contains a description of the USB or Serial protocols,
as well as further information known, about various glucometers,
whether they are on the market or not.

## License

This work is licensed under a
[Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

Use the information you'll find in this page for whatever purpose you
see fit, but there is no warranty about the accuracy of this
information.

Most of the information has been reverse engineerd by individual
contributors.

## Content

* Abbott Laboratories
  - [FreeStyle Optium](abbott/freestyle-optium.md)
  - [Shared HID protocol](abbott/shared-hid-protocol.md)
  - [FreeStyle Libre](abbott/freestyle-libre.md)
  - [FreeStyle Precision Neo](abbott/freestype-precision-neo.md)
* LifeScan
  - [OneTouch Verio (2015)](lifescan/onetouch-verio-2015.md)
* SD Biosensor
  - [SD Codefree](sd-biosensor/codefree.md)

## Structure

Please create a top-level directory for each of the manufacturers, and
place the protocol specifications within those directories. Use
Markdown format to describe the protocol, try to use
[ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_Form)
for the description of the protocol.
