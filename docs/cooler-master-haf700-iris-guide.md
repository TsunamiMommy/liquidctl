# Cooler Master HAF 700 EVO IRIS LCD

_Driver API and source code available in `liquidctl.driver.haf700_iris`._

_New in git._<br>

Support in this driver is for the original HAF 700 EVO IRIS device with USB ID
`2516:01c1`.  The panel is a 480×480 display backed by a small Android 4.4.2
computer.  The current liquidctl transport is Linux-only and requires the
system `adb` executable.

The driver controls only the IRIS display path.  It deliberately does not open
or claim the device's HID interface, so other software can continue to use that
interface independently.

## Requirements

The display must be visible to ADB before liquidctl can control it:

```console
$ adb devices -l
List of devices attached
1234567890ABCDEF    device ...
```

Install ADB with the distribution package manager.  Common package names are:

```console
# Fedora/Nobara
$ sudo dnf install android-tools

# Debian/Ubuntu
$ sudo apt install adb

# Arch Linux
$ sudo pacman -S android-tools
```

When the liquidctl udev rules are installed, `2516:01c1` is granted normal
seat-user access on Linux.  After installing or updating the rules, reload them
and reconnect the device if necessary.

## Initialization and status

Initialization verifies the ADB/TCP path and queries the IRIS media-storage
geometry:

```console
$ liquidctl --match "HAF 700 EVO IRIS" initialize
Cooler Master HAF 700 EVO IRIS
├── Native telemetry modes       13
├── LCD width                   480  px
├── LCD height                  480  px
├── Storage block size         4096  B
├── Storage block count       78936
└── Custom media mode            17
```

The storage values are obtained from the MagicBox `QUERY_SPACE` command and are
not needed for native telemetry pages; they are useful as an end-to-end
transport check.

## Native telemetry pages

The IRIS does not treat these as a single generic numeric screen.  The mode
selects the complete native presentation, including the CPU/GPU/fan label,
artwork, units, and layout.  The current value is supplied separately.

| liquidctl screen mode | IRIS mode | value unit | default range |
|---|---:|---|---:|
| `cpu-frequency` | 1 | MHz | 0–9999 |
| `gpu-frequency` | 2 | MHz | 0–9999 |
| `cpu-temperature` | 3 | °C | 0–100 |
| `gpu-temperature` | 4 | °C | 0–100 |
| `cpu-usage` | 5 | % | 0–100 |
| `gpu-usage` | 6 | % | 0–100 |
| `memory-usage` | 7 | % | 0–100 |
| `cpu-fan` | 8 | rpm | 0–9999 |
| `case-fan-1` | 9 | rpm | 0–9999 |
| `case-fan-2` | 10 | rpm | 0–9999 |
| `case-fan-3` | 11 | rpm | 0–9999 |
| `case-fan-4` | 12 | rpm | 0–9999 |
| `case-fan-5` | 13 | rpm | 0–9999 |

Frequency values are supplied in MHz.  MagicBox renders its frequency page in
GHz.  Temperature, utilization and RPM values are integer fields on the wire;
programmatic floating-point inputs are rounded to the nearest integer.

All thirteen native pages have been exercised on physical `2516:01c1`
hardware.

### Basic CLI examples

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-temperature 42
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen gpu-temperature 58
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-usage 37
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen gpu-usage 97
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen memory-usage 62
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-frequency 5200
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen gpu-frequency 2500
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-fan 1350
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen case-fan-5 950
```

Aliases such as `cpu-temp`, `gpu-temp`, `cpu_freq`, `gpu_freq`, `ram-usage`,
and `case-fan1` are accepted for convenience.

## Stateful select and update behavior

MagicBox uses two commands for telemetry pages:

- `0x12` (`LED_CAROUSEL`) selects/configures a page and returns an ACK;
- `0x15` (`RESPONSE_MODE_DATA`) updates an already selected page in place and
  intentionally returns no ACK.

With the default `action=auto`, a persistent driver instance sends `0x12` when
a page is first selected or when the mode changes, then uses `0x15` for later
value changes to the same mode:

```text
cpu-temperature = 42  -> 0x12, mode 3, value 42
cpu-temperature = 43  -> 0x15, mode 3, value 43
cpu-temperature = 44  -> 0x15, mode 3, value 44
gpu-temperature = 58  -> 0x12, mode 4, value 58
```

This is intended for long-running integrations such as monitoring daemons.  A
new standalone liquidctl CLI invocation creates a new driver instance, so it
normally starts with `0x12` again.

## Complete screen-value input

The full decoded MagicBox `LcdModeInstruction` is available through
`set_screen()`.  A caller can set every known field:

| field | accepted value | default |
|---|---|---|
| `value` | u16 numeric value | required |
| `minimum` | u16 | mode-specific |
| `maximum` | u16 | mode-specific |
| `scroll` | u8 | `0` |
| `reserved` | exactly 3 hexadecimal bytes | `000000` |
| `round-speed` | u8 | `0` |
| `text-color` | `RRGGBB` or `AARRGGBB` | `FFFFFFFF` |
| `text` | UTF-8, maximum 255 encoded bytes | empty |
| `background-state` | u8 | `0` |
| `background-color` | `RRGGBB` or `AARRGGBB` | `FF000000` |
| `image-name` | UTF-8, maximum 255 encoded bytes | empty |
| `media-name` | UTF-8, maximum 255 encoded bytes | empty |
| `action` | `auto`, `select`, or `update` | `auto` |

### Key/value CLI syntax

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-temperature \
  '42;minimum=0;maximum=100;scroll=0;reserved=000000;round-speed=0;text-color=FFFFFFFF;text=;background-state=0;background-color=FF000000;image-name=;media-name=;action=select'
```

Only fields that need to change must be supplied on a persistent connection.
Presentation fields from the selected page are retained by the driver.

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-temperature \
  '43;action=update'
```

`action=select` forces command `0x12`; `action=update` forces `0x15`.  These are
mainly useful for protocol testing.  Integrations should normally use `auto`.

### Positional CLI syntax

The positional field order is:

```text
value;minimum;maximum;scroll;reserved;round-speed;text-color;text;
background-state;background-color;image-name;media-name;action
```

For example:

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen cpu-temperature \
  '42;0;100;0;000000;0;FFFFFFFF;;0;FF000000;;;select'
```

### JSON syntax

JSON is convenient when string fields can contain punctuation:

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen gpu-temperature \
  '{"value":58,"minimum":0,"maximum":100,"text_color":"FFFFFFFF","background_color":"FF000000","action":"select"}'
```

## Custom media

MagicBox mode 17 displays user-provided media.  The driver supports PNG, JPEG
and MP4 files and has been physically validated with all three formats on the
tested `2516:01c1` panel.

```console
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen image ./panel.png
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen image ./panel.jpg
$ liquidctl --match "HAF 700 EVO IRIS" set lcd screen video ./panel.mp4
```

`image`, `media` and `video` select the same MagicBox media mode; the file
extension determines whether MagicBox builds an image or video view.  `.jpeg`
is accepted and uploaded with a `.jpg` suffix.

The driver hashes the file contents with SHA-256 and uploads it under a name of
the form:

```text
liquidctl/<sha256>.png
liquidctl/<sha256>.jpg
liquidctl/<sha256>.mp4
```

Content-addressed names are intentional.  MagicBox uses Glide for still images
and was observed to cache by pathname; reusing the same remote pathname for
different image bytes can display stale content.

### RAM-backed storage

Custom media is stored under:

```text
/data/data/com.magic.box/files/liquidctl
```

Before uploading media, the driver ensures that this dedicated subdirectory is
a 32 MiB `tmpfs`.  Individual media files are limited to 16 MiB.  The larger
RAM filesystem allows the currently displayed file and its replacement to
coexist while the new file is uploaded and selected.  The rest of MagicBox's
`files/` directory remains on its normal storage and is not hidden by the
mount.  The media payload therefore lives in RAM rather than being written to
the panel's UBIFS-backed `/data` filesystem.

The mount is intentionally left in place when liquidctl disconnects.  It is
volatile and disappears when Android reboots.  The next media request recreates
the mount, detects the missing content-addressed file with `FILE_EXIST`, uploads
it again, and selects mode 17.  This complete post-reboot recovery path has
been physically validated.

This only avoids the bulk media-file write.  MagicBox itself may still update
small application metadata such as preferences when a page is selected.

After a new media file has been selected successfully, the driver removes old
files from its isolated `liquidctl/` directory using MagicBox `DELETE_FILE`
with deletion state 1.  Physical testing established that state 1 deletes only
the named file; state 0 deletes all MagicBox files and is never used by the
driver.

Programmatic callers can disable stale-file cleanup for a request:

```python
dev.set_screen(
    "lcd",
    "image",
    {"path": "/path/to/panel.png", "cleanup": False},
)
```

## Programmatic use

Programmatic callers should pass a mapping rather than serialize parameters:

```python
dev.set_screen(
    "lcd",
    "gpu-temperature",
    {
        "value": 58.4,
        "minimum": 0,
        "maximum": 100,
        "text_color": "FFFFFFFF",
        "background_color": "FF000000",
        "action": "auto",
    },
)
```

For live updates, keep the same connected driver instance and send only the
changing value:

```python
dev.set_screen("lcd", "gpu-temperature", {"value": 59.1})
dev.set_screen("lcd", "gpu-temperature", {"value": 60.0})
dev.set_screen("lcd", "gpu-temperature", {"value": 61.2})
```

The driver preserves the page configuration and uses `0x15` for the subsequent
updates.

### Capability metadata

`Haf700Iris.get_screen_capabilities()` returns structured metadata intended for
persistent integrations.  It contains the channel, 480×480 resolution, all 13
native telemetry modes, their aliases/units/default ranges, the configurable
instruction fields, the select/update command IDs, and custom-media metadata
including mode 17, supported formats and the RAM-backed directory.

This method is a driver-specific extension rather than a standardized liquidctl
API.  It is provided so an integration can build its UI from the driver's
contract instead of duplicating HAF 700 protocol constants.

## Transport and reconnection

The driver uses the Android Debug Bridge only to reach MagicBox's TCP server:

```text
2516:01c1 interface 1 (ADB)
        |
        +-- adb forward tcp:0 tcp:9900
        |
        +-- localhost dynamically allocated port
        |
        +-- MagicBox TCP server on the IRIS
```

`tcp:0` lets ADB allocate a free local port, avoiding collisions and allowing
multiple devices to be addressed independently.

The driver does not call the normal `UsbDriver.connect()` implementation and
therefore does not detach or claim either USB interface.  The USB object is
used for liquidctl discovery and to obtain the serial number used to select the
matching ADB device.

If ADB or the TCP connection resets during a live `0x15` update, the driver:

1. closes the stale socket;
2. removes its old ADB forwarding rule;
3. waits for/rechecks the ADB device;
4. creates a new `tcp:0 -> tcp:9900` forward;
5. reconnects to MagicBox;
6. re-sends the last `0x12` page configuration;
7. retries the `0x15` update.

This recovery path has been physically tested by resetting the display/USB
connection while a page was being updated.  The same reconnect/reselect path
was also tested with mode 17 selected; the media remained visible after the
ADB/TCP forward was torn down and recreated.

## Limitations

- Only Linux has been validated and enabled by the upstream patch.
- The current transport depends on the external `adb` executable.
- Custom media relies on the panel's rooted ADB environment to create a
  dedicated tmpfs mount under MagicBox's private files directory.
- The original HAF 700 EVO with USB ID `2516:01c1` is the tested device.  Other
  revisions should not be assumed compatible solely from the product name.
