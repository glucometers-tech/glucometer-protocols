# FreeStyle InsuLinx

Reverse engineered by [Xavier Claessens](mailto:xclaesse@gmail.com).

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

The first field in the record specify the type of record; type 0 is a blood
glucose reading.

#### Blood glucose record fields

  1. `type = "0"`
  2. `id`
  3. `month = 1*2DIGIT`
  4. `day = 1*2DIGIT`
  5. `year = 1*2DIGIT`
  6. `hour = 1*2DIGIT`
  7. `minute = 1*2DIGIT`
  8. `unknown`
  9. `unknown`
  10. `unknown`
  11. `unknown`
  12. `unknown`
  13. `unknown`
  14. `value = 1*DIGIT`

      Unknown (but likely) whether this includes HI / LO constants.
  15. `unknown`
  16. `unknown`

### Other commands

The following commands appear to also be supported, but their meanings are not
known:

  * `$getrmndrst,0`
  * `$getrmndr,0`
  * `$rmdstrorder?`
  * `$actthm?`
  * `$wktrend?`
  * `$gunits?`
  * `$clktyp?`
  * `$alllang?`
  * `$lang?`
  * `$inslock?`
  * `$actinscal?`
  * `$iobstatus?`
  * `$foodunits?`
  * `$svgsdef?`
  * `$corsetup?`
  * `$insdose?`
  * `$inslog?`
  * `$inscalsetup?`
  * `$carbratio?`
  * `$svgsratio?`
  * `$mlcalget,3`
  * `$cttype?`
  * `$bgdrop?`
  * `$bgtrgt?`
  * `$bgtgrng?`
  * `$ntsound?`
  * `$btsound?`
  * `$custthm?`
  * `$taglang?`
  * `$tagsenbl?`
  * `$tagorder?`
  * `$result?`
  * `$gettags,2,2`
  * `$frststrt?`
