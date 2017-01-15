# Abbott Freestyle Libre HID protocol

Abbott devices such as FreeStyle InsuLinx and FreeStyle Libre share a
common communication protocol based on USB HID. The commands executed
on top of this protocol are not cross-device compatible.

See [Shared HID Protocol](shared-hid-protocol.md) for a description of
the framing.


## Communication Protocol

The "reports" are not following the HID standard. They are of a fixed
64-bytes size, even if most of the commands themselves are
shorter. Replies can span multiple inbound reports. Be aware that the
device does not clean up the buffer. After the length as indicated in
the header there is usually a lot of garbage that needs to be ignored.

## Initialization Sequence

See [Shared HID Protocol](shared-hid-protocol.md) for a description of
the Initialization sequence.

## Commands

Most of the commands are sent with command type 0x60.

The following commands are a subset of possible commands. To query
Diabetes relevant information, these are most important:

    cmd 0x60 msg $ptname?       ==> Name of Patient as provided in Freestyle Libre application
    cmd 0x60 msg $date?        ==> date: 12,8,16 (month, day, year)
    cmd 0x60 msg $time?        ==> time: 23,11 (hour, minute)
    cmd 0x21 msg $dbrnum?      ==> number of records
    cmd 0x60 msg $history?     ==> Sensor results
    cmd 0x60 msg $arresult?    ==> manual results, with comments

## Patient Name

The standard message framing is omitted for clarity. Insteas only the
command and the message part are outlined. The size can be calculated
accordingly. In the replies, also the type and length are
ommitted. Only the reply in the message itself is shown here.

    ptname: cmd = 0x60 msg = $ptname?
		ptname-reply: <name>\r\nCKSUM: <checksum>\r\nCMD: OK\r\n

## Date, Time

The standard message framing is omitted for clarity. Insteas only the
command and the message part are outlined. The size can be calculated
accordingly. In the replies, also the type and length are
ommitted. Only the reply in the message itself is shown here.

    date: cmd = 0x60 msg = $date?
    date-reply: mm,dd,yy\r\n
		CKSUM: <checksum>\r\n
		CMD: OK\r\n
		
		time: cmd = 0x60 msg = $time?
		time-reply: HH,MM\r\n
		CKSUM: <checksum>\r\n
		CMD: OK\r\n
		
## DB Records

The standard message framing is omitted for clarity. Insteas only the
command and the message part are outlined. The size can be calculated
accordingly. In the replies, also the type and length are
ommitted. Only the reply in the message itself is shown here.

    dbrecords: cmd = 0x21 msg = $dbrnum?
    dbrecords-reply: DBRECORDS = xxx\r\n
		CKSUM: <checksum>\r\n
		CMD: OK\r\n

## history

The standard message framing is omitted for clarity. Insteas only the
command and the message part are outlined. The size can be calculated
accordingly. In the replies, also the type and length are
ommitted. Only the reply in the message itself is shown here.

This command sends all measurements done by the Libre sensor. It does
not send measurements that are made by the patient. They are sent with
$arresult?

    history: cmd = 0x60 msg = $history?
    history-reply: lines of records, separated by \r\n
		CKSUM: <checksum>\r\n
		CMD: OK\r\n

A single result row can span multiple messages. A single message is
separated by \r\n. At the end of all records the cksum/cmd sequence is
sent.

The result line is a comma separated list. The known fields are as follows:
  0. ID of the record
  2. Month
  3. Day of the month
  4. Year - 2 digits!
  5. Hour
  6. Minutes
  7. Seconds
  13. Glucose value in mg/dL
  14. Runtime of the actual sensor in Minutes
  15. Some sort of validity check. If set to 32768, then ignore the value as the sensor is not ready

## arresult

The standard message framing is omitted for clarity. Insteas only the
command and the message part are outlined. The size can be calculated
accordingly. In the replies, also the type and length are
ommitted. Only the reply in the message itself is shown here.

    arrresult: cmd = 0x60 msg = $history?
    arresult-reply: lines of records, separated by \r\n
		CKSUM: <checksum>\r\n
		CMD: OK\r\n

A single result row can span multiple messages. A single message is
separated by `\r\n`. At the end of all records the cksum/cmd sequence
is sent.

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
