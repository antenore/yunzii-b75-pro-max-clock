# yunzii-b75-pro-max-clock

Sets the time on the display of the YUNZII B75 PRO MAX (USB `28e9:31c8`) from Linux. The manufacturer's tool is a web app (yunzii-game.com, WebHID, Chrome, Edge and Opera only) and it says Bluetooth is not supported. This script does the same from the terminal, with a systemd timer that keeps the time in sync.

Unofficial project, not affiliated with YUNZII. Tested on one keyboard only, firmware 1.09, connected by cable, on Manjaro. I don't know if it works over 2.4G and Bluetooth. Use at your own risk.

## Install

You need Python 3 (standard library only) and access to `/dev/hidraw`. The udev rule gives the ACL to the session user (`uaccess`), so no groups are needed.

```bash
sudo install -m755 yunzii-settime /usr/local/bin/
sudo install -m644 udev/50-yunzii-hidraw.rules /etc/udev/rules.d/
sudo udevadm control --reload && sudo udevadm trigger --subsystem-match=hidraw
install -Dm644 -t ~/.config/systemd/user systemd/yunzii-settime.service systemd/yunzii-settime.timer
systemctl --user daemon-reload && systemctl --user enable --now yunzii-settime.timer
```

If the keyboard does not get the permissions yet, unplug it and plug it back in.

## Usage

```bash
yunzii-settime                                # send the current time
yunzii-settime --dry-run                      # print the frames without sending them
yunzii-settime --at '2026-10-22 14:30:00'     # send a time of your choice
```

To test it, send a different time with `--at` and look at the display.

The timer is a user timer. It runs once a day (`OnCalendar=daily`) and once 2 minutes after the user manager starts. `Persistent=true` catches up missed runs and there is a random delay of up to 5 minutes. To change the schedule: `systemctl --user edit yunzii-settime.timer`.

Exit codes:

- 2: keyboard not connected. The service treats it as success, so no unit goes into a failed state
- 1: the keyboard answers wrongly or does not answer. The service shows as failed

## Protocol

Reconstructed by reading the public JavaScript of the manufacturer's tool.

It uses the VIA raw HID channel: usage page `0xFF60`, 64-byte report, that is the hidraw with that collection.

Frame:

- byte 0: command (`0x40` info, `0x41` data, `0x42` end)
- byte 3: data length
- byte 4-5: sum of the frame bytes, little endian
- from byte 7: the data

The info packet contains an internal command `A5 5A <cmd> 00 <n> <crc16>`. The CRC16 is Modbus (init `FFFF`, poly `A001`) computed over `[cmd, 0, n]`.

- time: cmd 9, data `[hour, minutes, seconds]`
- date: cmd 10, data `[year%100, day of the week (Sunday = 7), month, day]`

Everything is sent 3 times, as the site does. The keyboard answers every frame with the same command.

Display images and GIFs are not implemented (same protocol, I have not read it yet). The firmware update commands, present in the same protocol, are deliberately not implemented.

## License

GPL-3.0-or-later, see `LICENSE`. Author: Antenore Gatta.
