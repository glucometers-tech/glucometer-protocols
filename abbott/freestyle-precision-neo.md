# FreeStyle Precision Neo, Optium Neo, Optium Neo H

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.com).

## Important device notes

Abbott markets a number of similar meters with a different set of features
enabled for each in different markets. These differences appear to be
implemented similarly to the selection of glucose units used in
display/reporting.

The known devices sharing the same base protocol are the following:

  * **FreeStyle Precision Neo**, marketed in the United States;
  * **FreeStyle Optium Neo**, marketed in Europe;
  * **FreeStyle Optium Neo H**.

The FreeStyle Precision Neo (at the very least) can also read special β-ketones
testing strips.

### USB IDs

| Device                  | Vendor ID | Product ID |
| ---                     | ---       | ---        |
| FreeStyle Precision Neo | 1a61      | 3850       |

## Protocol

This device uses the [shared HID protocol](shared-hid-protocol.md) used by other
meters in the FreeStyle family. All messages that have been identified are
considered text commands and use message type `0x60`.

## Commands

All commands supported by the shared protocol are supported by the device. Those
that deviate from said protocol are here documented.

### `$result?`

The `$result?` command is used to dump the readings records from the device, and
it follows the *Multiple records command* output format as described in the
shared protocol documentation.

The first field in the record specify the type of record; type 7 is a blood
glucose reading, type 10 is an insulin input that was made on the device.


#### Blood glucose record fields

  1. `type = "7"`
  2. `id`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `unknown`
  9. `value = 1*DIGIT / "HI"`
  10. `unknown`
  11. `unknown`
  12. `unknown`
  13. `unknown`
  14. `unknown`
  15. `unknown`
  16. `unknown`
  17. `unknown`
  18. `unknown`
  19. `unknown`

The `HI` literal signifies a reading that was outside the range of the device 
(or more likely of the strip.) For glucose results, this is an unknown value.

#### Insulin record fields

  1. `type = "10"`
  2. `id`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `unknown`
  9.  `insulin-type`
  10. `value = 1*DIGIT`
  11. `unknown`
  12. `unknown`
  13. `unknown`

```
insulin-type = morning-long-acting / breakfast-short-acting /
               lunch-short-acting / evening-long-acting /
               dinner-short-acting
morning-long-acting = "0"
breakfast-short-acting = "1"
lunch-short-acting = "2"
evening-long-acting = "3"
dinner-short-acting = "4"
```

#### β-ketones record fields

  1. `type = "9"`
  2. `id`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `unknown`
  9. `value = 1*DIGIT`
  10. `unknown`
  
  The Ketone value is in mg/dL not in mmol/L as it should be. Divide the value by 18 to get a proper Ketone value in mmol/L.
