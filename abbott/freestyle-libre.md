# FreeStyle Libre

Reverse engineered by Pascal Fribi, editing and expansion by Diego Elio Pettenò.

## Important device notes

### USB IDs

| Device          | Vendor ID | Product ID |
| ---             | ---       | ---        |
| FreeStyle Libre | 1a61      | 3650       |

## Protocol

This device uses the [shared HID protocol](shared-hid-protocol) used by other
meters in the FreeStyle family. Most of the text command share the same message
type `0x60`, except where otherwise noted.

## Commands

The Libre supports a distinct set of commands from those described in the shared
protocol. In particular, the following commands are not supported:

  * `$serlnum?` — replaced by `$sn?`.

The following text commands are instead added:

  * `$dbrnum?`
  * `$history?`
  * `$arresult?`

## `$dbrnum?`

    dbrnum-response = "DBRECORDS = " 1*DIGIT CRLF

The response includes the number of records in the database (history results.)

## `$history?`

The `$history?` command returns all the automatic measurements taken by the
sensors. It does not include the immediate measurements on user request, nor
strip measurements (blood glucose and β-ketone).

The output follows the *Multiple record command* output format as described in
the shared protocol documentation.

### Record fields

  1. `recordid = 1*5DIGIT`
  2. `unknown`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `second = 1*2DIGIT`
  9. `unknown`
  10. `unknown`
  11. `unknown`
  12. `unknown`
  13. `value = 1*DIGIT`
  14. Runtime of the actual sensor in Minutes
  15. Some sort of validity check. If set to 32768, then ignore the value as the
      sensor is not ready

## `$arresult?`

The `$arresult?` returns manual measurements taken by the user, either by
scanning the sensor or using a testing strip (for either blood sugar or β-ketone
measurement)

The output follows the *Multiple record command* output format as described in
the shared protocol documentation.

Multiple record types have been identified; the second value in the record
identifies the type; type 2 is a manual reading, record type 5 identifies a time
change event.

### Reading record fields

  1. `record-id = 1*5DIGIT`
  2. `record-type = "2"`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `second = 1*2DIGIT`
  9. `unknown`
  10.
      ```
      reading-type = blood-glucose / blood-ketone / sensor-glucose
      blood-glucose = "0"
      blood-ketone = "1"
      sensor-glucose = "2"
      ```
  11. `unknown`
  13. `value = 1*DIGIT`

      When `reading-type` is either `blood-glucose` or `sensor-glucose`, this
      represent the blood sugar reading in mg/dL.

      When `reading-type` is `blood-ketone`, this represent the β-ketone
      reading, in an unknown measurement unit.
  14. `unknown`
  15. `unknown`
  16. `sports-flag = "0" / "1"`
  17. `medication-flag = "0" / "1"`
  18. `long-acting-insulin-flag = "0" / "1"`

      See field 24 for value.
  19. `short-acting-insulin-flag = "0" / "1"`

      See field 44 for value†.
  20. `custom-comments-bitfield = 1*DIGIT`

      Custom comments 1-6 flags. To be interpreted as a bitfield, LSB first.
  21. `unknown`
  22. `unknown`
  23. `unknown`
  24. Value of long Acting insulin in 0.5 IE. If you want proper IE, divide by 2
  25. `unknown`
  26. `food-flag = "0" / "1"`

      See field 27 for value.
  27. `food-carbs-grams = 1*DIGIT`
  28. `unknown`
  29. `unknown`
  30. Custom comment 1 ‡
  31. Custom comment 2 ‡
  32. Custom comment 3 ‡
  33. Custom comment 4 ‡
  34. Custom comment 5 ‡
  35. Custom comment 6 ‡
  36. `unknown` †
  37. `unknown` †
  38. `unknown` †
  39. `unknown` †
  40. `unknown` †
  41. `unknown` †
  42. `unknown` †
  43. `unknown` †
  44. Value of short acting insulin in 0.5 IE. If you want proper IE, divide by 2.

† The number of columns in a record can be different. If an short acting insulin
value has been set, then it is > 44, otherwise the record has only entries up to
comment 6.

‡ Be aware that custom comments are shown in every record. The bitfield in
field 19. shows which comments are actually set.


### Time change record fields

  1. `record-id = 1*5DIGIT`
  2. `record-type = "5"`
  3. `new-month = 1*2DIGIT`
  4. `new-day = 1*2DIGIT`
  5. `new-year = 1*2DIGIT`
  6. `new-hour = 1*2DIGIT`
  7. `new-minute = 1*2DIGIT`
  8. `new-second = 1*2DIGIT`
  9. `unknown`
  10. `old-month = 1*2DIGIT`
  11. `old-day = 1*2DIGIT`
  12. `old-year = 1*2DIGIT`
  13. `old-hour = 1*2DIGIT`
  14. `old-minute = 1*2DIGIT`
  15. `old-second = 1*2DIGIT`
  16. `unknown`
  17. `unknown`
  18. `unknown`
  19. `unknown`
  20. `unknown`
