<!--
SPDX-FileCopyrightText: 2016 The Glucometer Protocols Authors

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Glucometers Protocol Specifications

This website is intended to provide description of protocols (or "Computer
Interface Specifications") for various glucometers.

As few manufacturers publish documentation of their meters' protocols, the
majority of the content is produced by reverse engineering the data transmission
between meters and PCs, and documenting it.

## Reversed Engineered Protocols

* Abbott Laboratories
    - [FreeStyle Lite](abbott/freestyle-lite.md)
    - [FreeStyle Optium](abbott/freestyle-optium.md)
    - [Shared HID protocol](abbott/shared-hid-protocol.md)
    - [FreeStyle InsuLinx](abbott/freestyle-insulinx.md)
    - [FreeStyle Libre](abbott/freestyle-libre.md)
    - [FreeStyle Libre 2](abbott/freestyle-libre-2.md)
    - [FreeStyle Precision Neo](abbott/freestyle-precision-neo.md)
* LifeScan
    - [Shared Binary Protocol](lifescan/shared-binary-protocol.md)
    - [OneTouch Verio IQ](lifescan/onetouch-verio-iq.md)
    - [OneTouch Verio (2015)](lifescan/onetouch-verio-2015.md)
* Menarini
    - [GlucoMen areo](menarini/glucomen-areo.md)
* Sanofi
    - [BGStar and Mystar Extra](sanofi/bgstar-mystar.md)
* SD Biosensor
    - [SD Codefree](sd-biosensor/codefree.md)
* TaiDoc
    - [TD-42xx](taidoc/td42xx.md)

See [reverse engineering contribution
suggestions](contributing/reverse-engineered.md) for details on how to
contribute new protocols.

## Manufacturer Supplied Documentation

* [Ascensia Diabetes Care](http://protocols.ascensia.com/Programming-Guide.aspx)
  (_Contour_ brand, formerly under the Bayer name.)

## Standards and Common Protocols

### CLSI

[CLSI](https://clsi.org/) (Clinical and Laboratory Standards Institute) develop
and publish protocols that are used by some meter manufacturers.

The two main relevant documents are:

 * [Specification for Low-Level Protocol to Transfer Messages Between Clinical
   Laboratory Instruments and Computer Systems, 2nd
   Edition](https://clsi.org/standards/products/automation-and-informatics/documents/lis01/)
   (LIS01-A2)
 * [Specification for Transferring Information Between Clinical Laboratory
   Instruments and Information Systems, 2nd
   Edition](https://clsi.org/standards/products/automation-and-informatics/documents/lis02/)
   (LIS02-A2)

These are proprietary, not freely available standards.

### Bluetooth GATT

Many Bluetooth glucometers implement [GATT
Services](https://www.bluetooth.com/specifications/gatt/services/), which then
reference the following specifications:

 * [Glucose Profile (GLP)
   1.0](https://www.bluetooth.org/DocMan/handlers/DownloadDoc.ashx?doc_id=248025).
 * [Glucose Service (GLS)
   1.0](https://www.bluetooth.org/docman/handlers/downloaddoc.ashx?doc_id=248026).
 * [Continuous Glucose Monitoring Profile (CGMP)
   1.0.1](https://www.bluetooth.org/docman/handlers/downloaddoc.ashx?doc_id=310501)
 * [Continuous Glucose Monitoring Service (CGMS)
   1.0.1](https://www.bluetooth.org/docman/handlers/downloaddoc.ashx?doc_id=310502)

## License

This work is licensed under a [Creative Commons Attribution 4.0 International
License](https://creativecommons.org/licenses/by/4.0/).

Use the information you'll find in this page for whatever purpose you see fit,
but there is no warranty about the accuracy of this information.

Most of the information has been reverse engineerd by individual contributors,
see the individual page for details.
