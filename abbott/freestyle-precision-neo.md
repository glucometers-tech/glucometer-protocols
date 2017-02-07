# FreeStyle Precision Neo / FreeStyle Optium Neo

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.eu).

## Important device notes

The FreeStyle Precision Neo is a glucometer that can also read special β-ketones
testing strips. Unfortunately those strips are not available in Europe, so the
reverse engineering in this document is

The FreeStyle Precision Neo has the same protocol as the FreeStyle Optium
Neo. Although the devices are very much alike they appear to have different
feature sets.

FreeStyle Precision Neo is only available within the United States, while
FreeStyle Optium Neo is only available outside of the United States.

## Protocol

This device uses the [shared HID protocol](shared-hid-protocol) used by other
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
  9. `value = 1*DIGIT`
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

#### Insulin record fields

  1. `type = "10"`
  2. `id`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `unknown`
  9.  Carries an enumeration of the type of insulin used:
     ```
     insulinType = morning-long-acting / breakfast-short-acting /
                   lunch-short-acting / evening-long-acting /
                   dinner-short-acting
     morning-long-acting = "0"
     breakfast-short-acting = "1"
     lunch-short-acting = "2"
     evening-long-acting = "3"
     dinner-short-acting = "4"
     ```
  10. `value = 1*DIGIT`
  11. `unknown`
  12. `unknown`
  13. `unknown`
