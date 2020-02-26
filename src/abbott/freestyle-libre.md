# FreeStyle Libre

Reverse engineered by Pascal Fribi, editing and expansion by Diego Elio Pettenò.

## Important device notes

### USB IDs

| Device          | Vendor ID | Product ID |
| ---             | ---       | ---        |
| FreeStyle Libre | 1a61      | 3650       |

## Protocol

This device uses the [shared HID protocol](shared-hid-protocol.md) used by other
meters in the FreeStyle family. Text commands are sent by the original software
as message type `0x21`, with responses as `0x60`, but the device appears to
accept the commands as type `0x60` as well (compatible with other FreeStyle
devices).

## Commands

The Libre supports a distinct set of commands from those described in the shared
protocol. In particular, the following commands are not supported:

  * `$serlnum?` — replaced by `$sn?`.

A number of new text commands are added instead.

### `$dbrnum?`

    dbrnum-response = "DBRECORDS = " 1*DIGIT CRLF

The response includes the number of records in the database (history results.)

This number is permanent across factory resets.

### `$history?`

The `$history?` command returns all the automatic measurements taken by the
sensors. It does not include the immediate measurements on user request, nor
strip measurements (blood glucose and β-ketone).

The output follows the *Multiple record command* output format as described in
the shared protocol documentation.

#### Record fields

  1. `record-id = 1*5DIGIT`
  2. `unknown = "12"`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `second = 1*2DIGIT`
  9. `unknown = "1"`
  10. `unknown = "0"`
  11. `unknown = "0"`
  12. `unknown = "0"`
  13. `unknown = "0" / "1"`

    This value appears to only be 1 for the first reading of the sensor (with
    sensor runtime reporting 15 minutes).

  14. `value = 1*DIGIT`
  15. `sensor-runtime-minutes = 1*DIGIT`
  16. `error-bitfield = 1*5DIGIT`

    This field needs to be interpreted as a bitfield (in decimal
    representation). Flag 0x8000 indicates an error (invalid reading). If the
    error flag is set, the remaining bits refer to more error details that are
    not clear.

### `$arresult?`

The `$arresult?` returns manual measurements taken by the user, either by
scanning the sensor or using a testing strip (for either blood sugar or β-ketone
measurement)

The output follows the *Multiple record command* output format as described in
the shared protocol documentation.

Multiple record types have been identified; the second value in the record
identifies the type; type 2 is a manual reading, record type 5 identifies a time
change event.

#### Reading record fields

  1. `record-id = 1*5DIGIT`
  2. `record-type = "2"`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `second = 1*2DIGIT`
  9. `unknown = "1"`
  10. Identifies the type of reading in the record

        reading-type = blood-glucose / blood-ketone / sensor-glucose
        blood-glucose = "0"
        blood-ketone = "1"
        sensor-glucose = "2"

  11. `unknown = "0"`
  12. `unknown = "0" / "1"`

    This appears to be 1 when either an error is present, or when the read
    value is LO.

  13. `value = 1*DIGIT`

    When `reading-type` is either `blood-glucose` or `sensor-glucose`, this
    represent the blood sugar reading in mg/dL.

    When `reading-type` is `blood-ketone`, this represent the β-ketone reading
    in mmol/l after apply `value`/18. It seems that the value is reported in
    mg/dL and the conversion is using the wrong molar mass system for
    conversion.  But based on actual measurements the results are correct this
    way.

  14. `unknown = "0" / "1"`

    This appears to be 0 for values read from a blood strip, and 1 for values
    read from sensor. Appears redundant with the `reading-type` field.

  15. Corresponds to the arrow indicator in the UI.

      direction-indicator = none / down-fast / down / steady / up / up-fast
      none = "0"
      down-fast = "1"
      down = "2"
      steady = "3"
      up = "4"
      up-fast = "5"

  16. `sports-flag = "0" / "1"`
  17. `medication-flag = "0" / "1"`
  18. `rapid-acting-insulin-flag = "0" / "1"`

    See field 44 for value†.

  19. `long-acting-insulin-flag = "0" / "1"`

    See field 24 for value.

  20. `custom-comments-bitfield = 1*DIGIT`

    Custom comments 1-6 flags. To be interpreted as a bitfield, LSB first.

  21. `unknown = "0"`
  22. `unknown = "0"`
  23. `unknown = "0" / "1" / "2" / "3" / "4" / "5"`
  24. Value of long Acting insulin in 0.5 IE. If you want proper IE, divide by 2
  25. `unknown = "0" / "1"`
  26. `food-flag = "0" / "1"`

    See field 27 for value.

  27. `food-carbs-grams = 1*DIGIT`
  28. `unknown = "0"`
  29. `error-bitfield = 1*5DIGIT`

    This field needs to be interpreted as a bitfield (in decimal
    representation). Flag 0x8000 indicates an error (invalid reading). If the
    error flag is set, the remaining bits refer to more error details that are
    not clear.

    In the Event View on the device, these correspond to Err3 errors.

  30. `custom-comment-1 = DQUOTE *VCHAR DQUOTE` ‡
  31. `custom-comment-2 = DQUOTE *VCHAR DQUOTE` ‡
  32. `custom-comment-3 = DQUOTE *VCHAR DQUOTE` ‡
  33. `custom-comment-4 = DQUOTE *VCHAR DQUOTE` ‡
  34. `custom-comment-5 = DQUOTE *VCHAR DQUOTE` ‡
  35. `custom-comment-6 = DQUOTE *VCHAR DQUOTE` ‡
  36. `unknown` †
  37. `unknown` †
  38. `unknown` †
  39. `unknown` †
  40. `unknown` †
  41. `unknown` †
  42. `unknown` †
  43. `unknown` †
  44. Value of rapid acting insulin in 0.5 IE. If you want proper IE, divide by
      2.

† The number of columns in a record can be different. If a rapid acting insulin
value has been set, then it is > 44, otherwise the record has only entries up to
comment 6.

‡ Be aware that custom comments are shown in every record. The bitfield in
field 19. shows which comments are actually set.


#### Time change record fields

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

### `$swreset`

Restart the Libre device, not erasing any information.

    swreset-cmd = "$swreset"
    swreset-response = CRLF

### `$resetpatient`

This command completely resets the FreeStyle Libre reader, bringing it
effectively to factory reset.

CAUTION! This erases all information on the device, including currently-enabled
sensors.

While not accessible via the protocol, a reset does not erases the "Event Log"
as available on the device.

    resetpatient-cmd = "$resetpatient"
    resetpatient-response = *("Erasing Sector @ 0x" 6 HEXDIGIT CRLF)

### Sound Settings

It's possible to query and set the notification and touch tone settings of
FreeStyle Libre devices through two commands.

The commands have first been identified on [FreeStyle
Insulinx](freestyle-insulinx) but are not tested on it.

    notifications = notifications-enabled / notifications-disabled
    notifications-enabled = "1"
    notifications-disabled = "0"

    vibrate = vibrate-enabled / vibrate-disabled
    vibrate-enabled = "1"
    vibrate-disabled = "0"

    touch-tone = touch-tone-enabled / touch-tone-disabled
    touch-tone-enabled = "1"
    touch-tone-disabled = "0"

    volume = volume-high / volume-low
    volume-high = "1"
    volume-low = "0"

    ntsound-cmd = "$ntsound" ("?" / "," notifications "," vibrate)
    ntsound-query-response = notifications "," vibrate CRLF

    btsound-cmd = "$btsound" ("?" / "," touch-tone "," volume)
    btsound-query-response = touch-tone "," volume

### Reminders (`$getrmndr`)

It's possible to read the reminders status for the reader device.

The commands have first been identified on [FreeStyle
Insulinx](freestyle-insulinx) but are not tested on it.

    get-reminder-command = "$getrmndr,0"
    get-reminder-response = 12 (reminder-entry CRLF)
    reminder-entry = reminder-repeat ","
                     reminder-action ","
                     ( reminder-enabled / reminder-disabled ) ","
                     reminder-hour "," reminder-minute

    reminder-repeat = reminder-off / reminder-daily / reminder-timer
    reminer-off = "0"
    reminder-once = "1"
    reminder-daily = "2"
    reminder-timer = "3"

    reminder-action = reminder-glucose / reminder-insulin / reminder-alarm
    reminder-glucose = "10"
    reminder-insulin = "11"
    reminder-alarm = "12"

    reminder-enabled = "1"
    reminder-disabled = "0"

    reminder-hour = 1*2 DIGIT
    reminder-minute = 1*2 DIGIT

Note that while other `$getrmndr` arguments than 0 will be accepted by the
device, they return no data at all.

When the repeat is "off" (`0`), all other fields are also `0`.

When the repeat is timer, the hour/minutes are the repeat time from the
beginning of the reminder.

### Other Commands

The following commands are sent by various software used by the Libre, but their
output is not obvious.

#### `$brandname?`

Appears to report the brand name of the device, possibly differing among
marketing regions.

    brandname-cmd = "$brandname?"
    brandname-msg = "FreeStyle Libre" CRLF

#### `$uom?`

Unknown response. Best guess this _may_ be "Unit Of Measure".

    uom-cmd = "$uom?"
    uom-msg = "0" CRLF  ; on a mmol/l device

The theory of this being the unit of measure is supported by the `$gunits?`
command reporting `1` for a mg/dL FreeStyle Precision Neo reader.

#### `$marketlev?`

Unknown respoonse.

    marketlev-cmd = "$marketlev?"
    marketlev-msg = "1,0,0,0" CRLF

#### `$lang?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    lang-cmd = "$lang?"
    lang-msg = "6" CRLF  ; On an Engish (UK) unit.

#### `$iobstatus?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    iobstatus-cmd = "$iobstatus?"
    iobstatus-msg = "255,255" CRLF

#### `$foodunits?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    foodunits-cmd = "$foodunits?"
    foodunits-msg = "0" CRLF

#### `$svgsdef?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    svgsdef-cmd = "$svgsdef?"
    svgsdef-msg = "255" CRLF

#### `$corsetup?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    corsetup-cmd = "$corsetup?"
    corsetup-msg = "255" CRLF

#### `$insdose?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    insdose-cmd = "$insdose?"
    insdose-msg = "1" CRLF

#### `$inscalsetup?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    inscalsetup-cmd = "$inscalsetup?"
    inscalsetup-msg = "0,0" CRLF

#### `$carbratio?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    carbratio-cmd = "$carbratio?"
    carbratio-msg = "0,0" CRLF
                    "1,0" CRLF
                    "2,0" CRLF
                    "3,0" CRLF
                    "4,0" CRLF
                    "0" CRLF

#### `$svgsratio?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    svgsratio-cmd = "$svgsratio?"
    svgsratio-msg = "0,0" CRLF
                    "1,0" CRLF
                    "2,0" CRLF
                    "3,0" CRLF
                    "4,0" CRLF
                    "0" CRLF

#### `$cttype?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    cttype-cmd = "$cttype?"
    cttype-msg = "255" CRLF

#### `$bgdrop?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    bgdrop-cmd = "$bgdrop?"
    bgdrop-msg = "0,255" CRLF
                 "1,0" CRLF
                 "2,0" CRLF
                 "3,0" CRLF
                 "4,0" CRLF
                 "0" CRLF

#### `$bgtrgt?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    bgtrgt-cmd = "$bgtrgt?"
    bgtrgt-msg = "0,0" CRLF
                 "1,0" CRLF
                 "2,0" CRLF
                 "3,0" CRLF
                 "4,0" CRLF
                 "0" CRLF

#### `$bgtgrng?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    bgtgrng-cmd = "$bgtgrng?"
    bgtgrng-msg = "0,0,0" CRLF
                  "1,0,0" CRLF
                  "2,0,0" CRLF
                  "3,0,0" CRLF
                  "4,0,0" CRLF
                  "0" CRLF

#### `$tagsenbl?`

Originally identified on [FreeStyle Insulinx](freestyle-insulinx).

    tagsenbl-cmd = "$tagsenbl?"
    tagsenbl-msg = CRLF

#### `$patch?`

Return information about the sensors (patches) that the device initialized.

This is a multi-record result, but its format is not currently understood.
