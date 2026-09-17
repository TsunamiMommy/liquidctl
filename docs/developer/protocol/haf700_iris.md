# Cooler Master HAF 700 EVO IRIS protocol notes

These notes describe the portions of the original HAF 700 EVO IRIS protocol
used by the liquidctl driver.  The device tested identifies as USB
`2516:01c1`, product `HAF700`, revision `0.01`.

The protocol was established from the Android software present on the panel and
verified against physical hardware.  Unless otherwise stated, commands marked
"verified" below were sent to a real `2516:01c1` device and their visible or
returned behavior was confirmed.

## Hardware and USB layout

The tested device enumerates at USB Full Speed (12 Mbit/s) and exposes one
configuration containing two interfaces.

| interface | class/subclass/protocol | endpoints | purpose |
|---:|---|---|---|
| 0 | HID `03/00/00` | `0x81` interrupt IN, `0x01` interrupt OUT, 4 bytes | case/HID control; not used by this driver |
| 1 | vendor `ff/42/01` | `0x82` bulk IN, `0x02` bulk OUT, 64 bytes | Android Debug Bridge |

The `ff/42/01` interface tuple is the standard Android ADB USB interface form.
The liquidctl implementation intentionally leaves both interfaces unclaimed and
uses the host ADB client for interface 1.

Observed descriptors:

```text
VID:PID             2516:01c1
Manufacturer        Coolermaster
Product             HAF700
Serial              1234567890ABCDEF on tested unit
bcdDevice           0.01
USB speed           Full Speed (12 Mbit/s)
```

## Android environment

The tested panel reports:

```text
Android release     4.4.2
SDK                  19
ro.product.model     HID-compliant
adbd                  running as root
```

Relevant installed/running packages include:

```text
com.magic.box
com.rdamicro.pcdatareceiver
```

`com.magic.box` contains the IRIS presentation code and the TCP command server
used by the telemetry driver.  The latter listens on device TCP port 9900.

## Host transport

The current driver creates an ADB forward:

```text
adb -s <serial> forward tcp:0 tcp:9900
```

ADB returns the dynamically allocated local TCP port.  liquidctl connects to
that localhost port and speaks the MagicBox protocol directly.

ADB is a transport only; the MagicBox protocol itself is ordinary TCP and does
not contain ADB framing.

## MagicBox packet framing

Every packet starts with a six-byte header:

| offset | size | field |
|---:|---:|---|
| `0x00` | 1 | command |
| `0x01` | 3 | fixed sync bytes `80 00 01` |
| `0x04` | 2 | payload length, big-endian |
| `0x06` | N | payload |

A command with no payload is therefore:

```text
CC 80 00 01 00 00
```

where `CC` is the command byte.

## Commands

The following command IDs were recovered from MagicBox.  Only the commands
explicitly marked verified are relied on by the liquidctl driver.

| command | name | status / purpose |
|---:|---|---|
| `0x10` | `LED_DIRECTION` | recovered, not used by driver |
| `0x12` | `LED_CAROUSEL` | **verified**; select/configure native pages and mode 17 media; ACKs |
| `0x14` | `QUERY_MODE_DATA` | recovered, not used by driver |
| `0x15` | `RESPONSE_MODE_DATA` | **verified**; update native mode data in place; no ACK |
| `0x16` | `SAVE_DATE` | recovered, not used by driver |
| `0x17` | `GET_DATA` | recovered, file/data path |
| `0x18` | `SLEEP_TYPE` | recovered, not used by driver |
| `0x19` | `GET_VERSION` | recovered, not used by driver |
| `0x1a` | `SET_VOLUME` | recovered, not used by driver |
| `0xf0` | `QUERY_SPACE` | **verified**; returns storage geometry |
| `0xf1` | `QUERY_FILE_LIST` | **verified**; top-level filesDir listing; not used for driver cleanup |
| `0xf2` | `DELETE_FILE` | **verified**; state 1 deletes one named file; state 0 deletes all |
| `0xf3` | `FILE_EXIST` | **verified**; tests one filename/path |
| `0xf4` | `PLAY_FILE` | recovered; no-op in the tested MagicBox build |
| `0xf5` | `JOIN_DOWNLOAD` | **verified**; begin upload and open target file |
| `0xf6` | `EXIT_DOWNLOAD` | **verified**; close upload stream |
| `0xf7` | `DOWNLOAD_FILE` | **verified**; append upload chunk (first 4 bytes ignored by build) |
| `0xf8` | `LED_CONTROL` | recovered, LCD state/control |

### `QUERY_SPACE` (`0xf0`)

Request:

```text
f0 80 00 01 00 00
```

Verified response on the test panel:

```text
f0 80 00 01 00 08 00 00 10 00 00 01 34 58
```

The eight-byte payload contains two big-endian u32 values:

```text
block_size   = 4096
block_count  = 78936
```

### File-transfer commands (`0xf2`, `0xf3`, `0xf5`-`0xf7`)

These commands were physically validated while adding custom-media support.
MagicBox stores relative names below its application `filesDir()`.  Relative
subpaths work, so a media name such as `liquidctl/<hash>.png` resolves below
`files/liquidctl/`.

`FILE_EXIST` (`0xf3`) takes the UTF-8 filename as its request body.  The response
payload is one status byte followed by the same filename:

```text
01 <filename>     exists
00 <filename>     absent
```

`JOIN_DOWNLOAD` (`0xf5`) begins a transfer.  Its request body is a big-endian
u32 file size followed immediately by the UTF-8 filename.  A successful response
starts with `00 00 00 00` and then echoes the filename; an invalid request uses
`ff ff ff ff`.

`DOWNLOAD_FILE` (`0xf7`) carries a four-byte prefix followed by file bytes.  In
the tested MagicBox build the first four bytes are ignored and only bytes 4..N
are written to the open stream.  The driver sends the big-endian file offset in
that prefix and uses 16 KiB chunks.  The observed ACK body is:

```text
00 00 00 04
```

`EXIT_DOWNLOAD` (`0xf6`) flushes/closes the upload stream and responds with a
zero-length payload.

`DELETE_FILE` (`0xf2`) takes one state byte followed by the UTF-8 filename.
Physical testing corrected an ambiguity in the decompiled code:

```text
state 1   delete only the named file
state 0   delete all files in MagicBox filesDir()
```

The response echoes the request body.  The liquidctl driver only uses state 1.
State 0 is deliberately never used.

`QUERY_FILE_LIST` (`0xf1`) was also tested.  Its response is a `|`-separated
listing of the top-level application files directory.  It returns `liquidctl`
for the driver's subdirectory but does not recurse into it, so the driver lists
its isolated tmpfs directory through ADB when deciding which stale files to
delete.

## `LcdModeInstruction` payload

Both `LED_CAROUSEL` (`0x12`) and `RESPONSE_MODE_DATA` (`0x15`) carry the same
variable-length `LcdModeInstruction` body.

| offset | size | field | encoding |
|---:|---:|---|---|
| `0x00` | 1 | mode | u8 |
| `0x01` | 2 | current value | big-endian u16 |
| `0x03` | 2 | minimum | big-endian u16 |
| `0x05` | 2 | maximum | big-endian u16 |
| `0x07` | 1 | scroll | u8 |
| `0x08` | 3 | reserved | raw bytes |
| `0x0b` | 1 | round speed | u8 |
| `0x0c` | 4 | text color | big-endian AARRGGBB |
| `0x10` | 1 | text byte length T | u8 |
| `0x11` | T | text | UTF-8 bytes |
| `0x11+T` | 1 | background state | u8 |
| `0x12+T` | 4 | background color | big-endian AARRGGBB |
| `0x16+T` | 1 | image-name byte length I | u8 |
| `0x17+T` | I | image name | UTF-8 bytes |
| `0x17+T+I` | 1 | media-name byte length M | u8 |
| `0x18+T+I` | M | media name | UTF-8 bytes |

Total payload size is `24 + T + I + M` bytes.

The driver exposes every recovered field even though ordinary telemetry updates
normally need only mode and current value.

### Known parser caveat

The decompiled MagicBox implementation appears to parse the low byte of the
minimum and maximum fields without consistently masking it to unsigned before
combining the two bytes.  The default ranges used by the driver avoid values
that expose this ambiguity.  Callers using custom ranges should treat unusual
values with low-byte bit 7 set as not yet fully characterized.

## Native telemetry modes

The mode number selects a complete MagicBox presentation, not only a unit or
sensor type.  CPU and GPU temperature, for example, use different labels and
artwork even when their numeric values are identical.

| ID | liquidctl name | physical presentation | value |
|---:|---|---|---|
| 1 | `cpu-frequency` | CPU frequency page | MHz; MagicBox renders GHz |
| 2 | `gpu-frequency` | GPU frequency page | MHz; MagicBox renders GHz |
| 3 | `cpu-temperature` | CPU temperature page | °C |
| 4 | `gpu-temperature` | GPU temperature page | °C |
| 5 | `cpu-usage` | CPU utilization page | 0–100 % |
| 6 | `gpu-usage` | GPU utilization page | 0–100 % |
| 7 | `memory-usage` | memory utilization page | 0–100 % |
| 8 | `cpu-fan` | CPU Fan page | rpm |
| 9 | `case-fan-1` | Case Fan 1 page | rpm |
| 10 | `case-fan-2` | Case Fan 2 page | rpm |
| 11 | `case-fan-3` | Case Fan 3 page | rpm |
| 12 | `case-fan-4` | Case Fan 4 page | rpm |
| 13 | `case-fan-5` | Case Fan 5 page | rpm |

All thirteen page selections were physically verified.

MagicBox also contains non-telemetry presentation modes.  Mode 17 is the
custom-media view and has been physically validated by liquidctl with PNG, JPEG
and MP4 files.  The exact behavior and numbering of the other non-telemetry
modes are not relied on by this driver.

For mode 17, MagicBox reads `media_name`, resolves it below `getFilesDir()`, and
selects an image view for `.png`/`.jpg` or a video view for `.mp4`.  `PLAY_FILE`
(`0xf4`) is not needed; selecting mode 17 with `media_name` is sufficient.


## RAM-backed custom-media storage

The panel's `/data` filesystem is UBIFS on UBI/raw NAND.  To avoid repeatedly
writing media payloads to flash, liquidctl reserves this application subdirectory:

```text
/data/data/com.magic.box/files/liquidctl
```

and mounts a 32 MiB tmpfs on that directory through the already-root ADB shell.
Individual media files are limited to 16 MiB, allowing the currently displayed
file and its replacement to coexist during an update.  Only that dedicated
subdirectory is shadowed; MagicBox's other files remain on the normal
filesystem.  A one-time `mkdir` may create the empty mountpoint on flash, but
subsequent media payload bytes are written to RAM.

The driver uses SHA-256 content-addressed filenames because physical testing
showed that Glide can display stale still-image content when different bytes are
reuploaded under the same pathname.  A new content hash produces a new remote
pathname and avoids that cache aliasing.

The tmpfs disappears on Android reboot.  On the next media request the driver
recreates the mount, observes that the hash-named file is absent, reuploads it,
and reselects mode 17.  This was tested after both manual tmpfs unmount and a
full `adb reboot`.

## Select versus update

`LED_CAROUSEL` (`0x12`) selects/configures the page.  A successful request is
acknowledged by a zero-length `0x12` response:

```text
12 80 00 01 00 00
```

`RESPONSE_MODE_DATA` (`0x15`) changes the data shown by the selected page
without visibly reloading the presentation and does not return an ACK.

Physical test sequence:

```text
0x12 mode=3 value=42 -> CPU Temperature page displays 42 °C
0x15 mode=3 value=50 -> same page updates to 50 °C
0x15 mode=3 value=60 -> same page updates to 60 °C
0x15 mode=3 value=70 -> same page updates to 70 °C
0x15 mode=3 value=45 -> same page updates to 45 °C
```

This behavior is the basis of the driver's persistent `action=auto` semantics.

## Verified CPU-temperature packet

A 42 °C CPU-temperature page using the default presentation fields is:

```text
12 80 00 01 00 18
03 00 2a 00 00 00 64 00
00 00 00 00
ff ff ff ff
00
00
ff 00 00 00
00
00
```

Decoded:

```text
command              12          LED_CAROUSEL
payload length       0018        24 bytes
mode                 03          CPU temperature
current value        002a        42
minimum              0000        0
maximum              0064        100
scroll               00
reserved             000000
round speed          00
text color           ffffffff
text length          00
background state     00
background color     ff000000
image-name length    00
media-name length    00
```

## Reconnection behavior

USB/ADB resets were observed during earlier VM passthrough experiments and are
treated as normal recoverable transport failures by the driver.

For a failed live update, the implementation recreates the ADB forward and TCP
connection, re-sends the last known `0x12` selection, then retries the `0x15`
update.  This sequence was tested on the physical device and restored the page
without requiring a process restart.  The same restoration path was tested with
mode 17 selected; reconnecting and re-sending the saved media instruction kept
the image visible.

## Implementation boundary

The current liquidctl driver owns:

- USB VID/PID discovery;
- mapping the USB serial to the ADB device;
- dynamic ADB port forwarding;
- TCP connection management;
- MagicBox packet framing;
- all 13 native telemetry mode IDs and defaults;
- `LcdModeInstruction` serialization;
- `0x12`/`0x15` state management;
- verified mode 17 PNG/JPEG/MP4 media presentation;
- `0xf2`/`0xf3`/`0xf5`/`0xf6`/`0xf7` media-file management;
- isolated RAM-backed media storage and stale-file cleanup;
- reconnect and page restoration.

Higher-level software only needs to choose a native presentation mode and
provide the current numeric metric.  It does not need to know command IDs,
MagicBox port numbers, ADB details, or IRIS mode numbers.
