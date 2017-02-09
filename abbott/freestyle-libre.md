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

  * `$dbrnum?` (as message type `0x21`)
  * `$history?`
  * `$arresult?`

## `$dbrnum?`

This is the only command that uses message type `0x21`.

    dbrnum-response = "DBRECORDS = " 1*DIGIT CRLF

The response includes the number of records in the database (history results.)

## `history?`

The `$history?` command returns all the automatic measurements taken by the
sensors. It does not include the immediate measurements on user request, nor
strip measurements (blood glucose and β-ketone).

The output follows the *Multiple record command* output format as described in
the shared protocol documentation.

### Record fields

  1. ID of the record
  2. Unknown
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

The result line is a comma separated list. The known fields are as follows:
  0. ID of the record
  2. Month
  3. Day of the month
  4. Year - 2 digits!
  5. Hour
  6. Minutes
  7. Seconds
  12. Glucose value in mg/dL
  15. Indication that sports have been recorded. 1 if Sports set, 0 otherwise. No further info on what sport and how long
  16. Indication that a medication has been administered. 1 if administered, 0 otherwise. No further info on what medication.
  17. Set to 1 if in another field the long acting insulin is set
  18. Set to 1 if in another field the short acting insulin is set
  19. Bitfield for custom comments 1-6. If 1, then first comment is set, if 2 then second comment (==> 3 = comment 1 and 2)...
  23. Value of long Acting insulin in 0.5 IE. If you want proper IE, divide by 2
  25. Indication that Carbs have been entered. 1 if carbs entered, 0 otherwise.
  26. Entered carbs in gramms
  29-34. Custom comments 1-6. Be aware that the comments are shown in every record. The bitfield in column 19. shows which comments are actually set.
  43. Value of short acting insulin in 0.5 IE. If you want proper IE, divide by 2.
	
The number of columns in a record can be different. If an short acting
insulin value has been set, then it is > 43, otherwise the record has
only entries up to comment 6.
