# Hacking the YUNZII B75 PRO MAX

Everything found so far about the firmware and the display protocol of the YUNZII B75 PRO MAX, enough to implement image and GIF upload. The README covers the clock; this file covers the rest.

Every claim has a source:

- **Device**: verified on the keyboard (one unit, firmware bcdDevice 1.09, USB cable, Manjaro Linux).
- **Vendor JS**: read from the JavaScript of the vendor tool (yunzii-game.com). Not run against the device.
- **Guess**: not verified.

## How the information was gathered

On the device:

- USB and HID report descriptors from sysfs and `lsusb`.
- Only VIA read commands: protocol version, keyboard value, layer count, macro count and size, custom get value, keymap buffer.
- One normal time update, with the replies printed.
- No keymap writes, no EEPROM reset, no bootloader jump. The keyboard was not opened.

From the vendor tool: the JavaScript bundle of yunzii-game.com, a single file `assets/index-<hash>.js` of about 2.4 MB, readable after prettier.

The server sends only its leaf certificate, without the RapidSSL TLS RSA CA G1 intermediate. So curl fails with "unable to get local issuer certificate", while browsers work because they fetch the intermediate via AIA. To fix it without `-k`:

1. Download the intermediate from the AIA URL in the leaf: `http://cacerts.rapidssl.com/RapidSSLTLSRSACAG1.crt`.
2. Convert it from DER to PEM.
3. Append it to the system CA bundle and pass that file with `--cacert`.

## USB

| Item | Value |
|---|---|
| VID:PID | `28e9:31c8` |
| Manufacturer | `SmartCloud` |
| Product | `YUNZII B75 PRO MAX Keyboard` |
| bcdDevice | 1.09 |
| Serial | none |
| Speed, power | full speed, 500 mA |
| Interfaces | 4 |

`28e9` is GigaDevice, so the MCU is very likely a GD32. Guess, the keyboard was not opened.

| Interface | Function | Endpoints |
|---|---|---|
| 0 | Boot keyboard | EP `0x81` IN, 8 bytes |
| 1 | VIA raw HID, usage page `0xFF60`, usage `0x61` | EP `0x82` IN and EP `0x03` OUT, 64 bytes |
| 2 | QMK shared endpoint. Report IDs: 2 mouse, 3 system control, 4 consumer, 6 NKRO keyboard (bitmap) | EP `0x84` IN, 32 bytes |
| 3 | Usage page `0xFF31`, usage `0x74`, input only. This is the QMK console (`CONSOLE_ENABLE`, what `hid_listen` and `qmk console` read) | 32 bytes |

All device-verified. Interface 1 is the channel the vendor site and `yunzii-settime` use. On interface 3 nothing came in 11 seconds, so debug output is off.

On Linux there are 4 hidraw nodes. The right one is the node whose report descriptor starts with `06 60 ff`.

## Firmware

It is QMK with VIA. All of this is device-verified.

- VIA protocol version `0x000C` (12), so QMK 0.19 or newer (new keycode format).
- Keyboard value uptime works (milliseconds). Layout options: 0.
- Switch matrix state (`0x02 0x03`) answers, so the VIA matrix tester should work.
- Firmware version (`0x02 0x04`) returns 0.
- 4 dynamic layers. 16 macros, macro buffer 3306 bytes (`0x0CEA`).
- Matrix 6x15.
- RGB matrix on VIA custom channel 3. Values read at the time: brightness `0xC0`, effect 4, speed `0x7F`, hue `0x58`, saturation `0xFF`.
- VIA custom channel 0 answers `0xFF` (unhandled).

### Custom keycodes

Custom keycodes are QMK `QK_KB_n` = `0x7E00 + n`. The vendor VIA JSON (`YUNZII_B75_PRO_MAX_VIA_V0102_250829.json`) lists 17 of them, `0x7E00` to `0x7E10`, in this order:

2.4G, BLE1, BLE2, BLE3, Mission Control, Launchpad, Left Option, Left Cmd, Right Cmd, Toggle Mac, Toggle Windows, Toggle Lock Win, Reset ("Mcu Reset"), Battery Level, Rgblight Mode, Rgblight Brightness+, Rgblight Brightness-.

## Keymap as read from the keyboard

Layers:

| Layer | Role |
|---|---|
| 0 | Windows base |
| 1 | Windows Fn layer. The Fn key at 5,10 is `MO(1)` = `0x5221` |
| 2 | Mac base. Left Option `0x7E06`, Left Cmd `0x7E07`, Right Cmd `0x7E08` on the modifier row |
| 3 | Mac Fn layer. `MO(3)` = `0x5223` |

Other keys:

- Knob push at 0,14 is `KC_MUTE` (`0x00A8`).
- Right column from the top: Delete (1,14), End (2,14), Page Up (3,14), Page Down (4,14).
- Unused positions on the Fn layers read `0x0001` (`KC_TRNS`).

### Fn layer 1 (Windows)

- F1 brightness down, F2 brightness up, F3 Win+Tab (`0x082B`), F4 My Computer, F5 Mail, F6 WWW Home, F7 previous track, F8 play/pause, F9 next track, F10 Home, F11 Insert, F12 Print Screen.
- 1, 2, 3: Bluetooth host 1, 2, 3 (`0x7E01` to `0x7E03`). 4: 2.4G (`0x7E00`).
- Backspace RGB toggle (`0x7820`), backslash RGB mode next (`0x7821`), Enter hue up (`0x7823`), Up value up (`0x7827`), Down value down (`0x7828`), Right speed up (`0x7829`), Left speed down (`0x782A`).
- S Toggle Mac (`0x7E09`), left Win Toggle Lock Win (`0x7E0B`), Space Reset (`0x7E0C`), right Ctrl Battery Level (`0x7E0D`).
- Six keys with unknown function:

  | Key | Keycode |
  |---|---|
  | Tab | `0x7E11` |
  | 8 | `0x7E17` |
  | 9 | `0x7E18` |
  | 0 | `0x7E19` |
  | Delete | `0x7E1A` |
  | End | `0x7E1B` |

  They are not in the VIA JSON, so VIA shows them as unknown. Guess: display or side-light controls. Check the manual or press them.

### Fn layer 3 (Mac)

- F3 Mission Control (`0x00C1`), F4 Launchpad (`0x00C2`), F10 mute, F11 volume down, F12 volume up.
- A is Toggle Windows (`0x7E0A`).
- The rest is the same as layer 1, without Lock Win and Toggle Mac.

## Problems in the vendor VIA JSON (V0102_250829)

- The six custom keycodes above (`0x7E11`, `0x7E17` to `0x7E1B`) are missing.
- The JSON puts Enter at matrix position 3,12. The keymap has `KC_ENTER` at 3,13 and backslash (`0x0031`) at 3,12, the ISO position. So remapping Enter in VIA probably changes the wrong key. Not confirmed yet with the matrix tester.
- The JSON is ANSI only. Position 4,1 holds `KC_NUBS` (`0x0064`, the ISO key) and is not in the layout.
- The Color menu entry has a `showIf` that excludes effects 24, 28, 29 and 32, but the dropdown has only 20 effects (0 to 19). It looks copied from another board. Harmless.

## Display protocol

Sources: the time and date commands and the reply format are device-verified. Everything else in this section is from the vendor JS and has not been sent to the keyboard.

### Outer frame

64 bytes on the VIA channel, report ID 0.

| Byte | Meaning |
|---|---|
| 0 | Command: `0x40` info, `0x41` data, `0x42` end |
| 1-2 | Offset of this chunk, little endian. 0 for the first. The vendor tool splits a payload in chunks of 57 bytes (64-7) and puts `chunk_index * 57` here |
| 3 | Number of data bytes in this frame: 57 for full chunks, the remainder for the last one. The end frame `0x42` carries 57 with no data |
| 4-5 | 16-bit sum of all frame bytes, computed with bytes 4-5 at zero, little endian |
| 6 | 0 when sending |
| 7-63 | Data |

Reply (device-verified): the keyboard echoes the frame with byte 6 set to `0x55`. The vendor tool checks only bytes 0 and 1. Example:

```
tx  40 00 00 07 f6 02 00 a5 5a 09 00 03 ...
rx  40 00 00 07 f6 02 55 a5 5a 09 00 03 c3 e1 ...
```

The payload of the time command, `[hour, minute, second]`, goes in the next `0x41` frame.

### Inner header

```
A5 5A <cmd> <lenHi> <lenLo> <crcHi> <crcLo>
```

The payload follows the header.

- CRC16 Modbus (init `FFFF`, poly `A001`) over `[cmd, lenHi, lenLo]` only, not over the payload.
- The length is big endian. The README writes it as `00 <n>`, which is the same for short payloads.
- For the 30720-byte image the header is `A5 5A 0C 78 00 <crc>`.

### Inner commands

Only 9 and 10 have been used on the device. The rest is from the vendor JS.

| cmd | Meaning | Payload |
|---|---|---|
| 9 | Set time | hour, minute, second |
| 10 | Set date | year%100, weekday (Sunday=7), month, day |
| 11 | Show home page | none |
| 12 | Image pixel data follows | length = width * height * 2 |
| 13 | Show image page | none |
| 14 | Delete image | none. The "clear images" button sends it 16 times |
| 15 | Show GIF page | none |
| 16 | Select target | `[1]` before an image. `[2, mode, frame_index]` before each GIF frame |
| 17 | GIF frame data follows | length = frame size |
| 18 | GIF frame count | `[mode, count]` |
| 19 | GIF speed | `[mode, fps]`, fps 1 to 60, default 30 |

### Pixels

The display is 160x96. Each pixel is RGB565, `(r>>3)<<11 | (g>>2)<<5 | (b>>3)`, sent high byte first. Rows go top to bottom. A full image is 160 * 96 * 2 = 30720 bytes.

### Upload an image (vendor sequence)

1. `0x40` info with `A5 5A 10 00 01 <crc> 01` (select image).
2. Wait 300 ms.
3. `0x41` data with the header `A5 5A 0C 78 00 <crc>`.
4. `0x41` data with the 30720 pixel bytes, in 57-byte chunks, offset = `chunk_index * 57`.
5. `0x42` end.

To delete: `0x40` with cmd 14, then `0x42`.

### Upload a GIF (vendor sequence)

Modes:

| Mode | Target | Max frames | Frame size |
|---|---|---|---|
| 0 | GIF page | 64 | 160x96 |
| 1 | Saved to the home page | 160 | 160x96 |
| 2 | Boot animation | 42 | 96x64 (12288 bytes per frame) |

Sequence:

1. `0x40` info with cmd 18 `[mode, 0]`.
2. `0x41` data with cmd 19 `[mode, 0]`.
3. Wait 30 ms.
4. For each frame `i`:
   1. `0x41` data with cmd 16 `[2, mode, i]`.
   2. Wait 3 s if `i` is a multiple of 16 (frame 0 included), otherwise 30 ms.
   3. `0x41` data with the cmd 17 header (length = frame size).
   4. Send the frame bytes in pieces of 1024 bytes. Each piece is its own run of `0x41` frames, with offsets starting again from 0. Wait 30 ms between pieces.
5. `0x41` data with cmd 18 `[mode, frame_count]`, wait 30 ms.
6. `0x41` data with cmd 19 `[mode, fps]`, wait 30 ms.
7. `0x42` end.

In the GIF flow almost everything goes through `0x41` data frames, with one `0x42` at the end. The inner `A5 5A` headers travel inside the data stream.

To clear GIFs: cmd 18 `[1, 0]` then cmd 19 `[1, 0]`, each as `0x40` + `0x42`. The vendor code declares length 1 for cmd 18 there but sends two bytes.

### Guess

The `A5 5A` framing with a CRC on the header only, and headers travelling inside the data stream, look like QMK forwarding the bytes over UART to a second chip that drives the display (maybe the wireless module). The 3 s pause every 16 GIF frames looks like flash erase time. Confirming this needs the keyboard opened.

## The vendor web tool

- It uses WebHID, picks the HID collection with usage `0x61` for these models and sends with report ID 0.
- Models handled this way (QMK + VIA + screen, the list is called `Bc` in the code), vendor `0x28E9`:

  | PID | Screen |
  |---|---|
  | `0x30AF` | 96x160 |
  | `0x3101` | 160x96 |
  | `0x3192` | 128x128 |
  | `0x31B4` | 240x135 |
  | `0x31BC` | 128x128 |
  | `0x31C8` | 160x96 (this keyboard) |

  The same protocol probably works on the others with a different size. Untested.
- For these models the site skips its whole proprietary configuration protocol (key mapping, lighting, profiles, firmware update). It shows only the "Screen" page: switch to home, image or GIF page, update time, clear images, clear GIF, upload image, upload GIF (GIF page, home page or boot animation, frame rate slider 1 to 60).
- The site's other keyboard family ("GamingKeyboard2" in the code) uses byte-0 opcodes `0x10` to `0x3B`, `0x55` and `0xB0` to `0xB7` (firmware version, bootloader, upgrade). On this keyboard `0x10` to `0x15` are VIA commands. For example `0x10` is `dynamic_keymap_macro_reset` (clears all macros) and `0x13` writes the keymap buffer. Do not replay them here.

## Do not send without a plan

- VIA `0x0A` (EEPROM reset) and `0x0B` (bootloader jump).
- Any VIA set command, unless you mean it.
- The opcodes of the other vendor family above, and `0xB0` to `0xB7`.
- Unknown commands in general. No fuzzing: on other models the same channel has firmware update paths.

## Open questions and next steps

- Ask YUNZII for the source. QMK is GPL-2.0-or-later, so whoever ships the binary has to provide the source on request.
- Confirm the Enter position with the VIA matrix tester: hold Enter and read the switch matrix state.
- Find what `0x7E11` and `0x7E17` to `0x7E1B` do. Then they can be added to the JSON.
- Implement image and GIF upload in this repo. The protocol above is complete enough.
- Open the keyboard: identify the MCU, display controller, SPI flash, wireless module and SWD pads.
- The bootloader VID:PID would identify the MCU family. Entering the bootloader (VIA `0x0B`, or the Reset keycode, which may be just a reboot) is a risk on a custom bootloader. Not done.
- The QMK console is compiled in. Assigning the `DB_TOGG` keycode to a spare key via VIA might turn on debug output on interface 3. Untested.
- Does the display protocol work over 2.4G or Bluetooth? Unknown. The site says Bluetooth is not supported.
- Time and date replies carry no extra data. It is unknown whether any inner command returns data. The vendor code copies reply bytes 7 and up, but nothing uses them for the screen.
