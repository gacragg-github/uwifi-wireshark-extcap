# uwifi-wireshark-extcap

Wireshark extcap interfaces for USB Wi-Fi adapters on macOS, built on
[userspace-wifi](https://github.com/kismetwireless/userspace-wifi) (`uwifi`).
Each USB adapter that `uwifi` supports appears as its own capture interface in
Wireshark, with a per-adapter channel dropdown, so you can capture 802.11
monitor-mode traffic from the Wireshark start page without a terminal.

## What it does

- Lists every adapter shown by `uwifi list` as a separate Wireshark interface
  (for example `mt7921-0`, `mt7925-0`), labelled with its driver, USB ID and speed.
- Builds each adapter's channel dropdown from `uwifi info`, so an adapter only
  offers bands and widths it reports: 20MHz channels, 40/80/160MHz combinations,
  and hop presets across 2.4, 5 and 6GHz.
- Accepts any channel list in `uwifi` notation in a custom field.
- Has a reload button that re-reads an adapter's supported channels.
- Streams radiotap pcap data from `uwifi` into Wireshark. Select several
  adapters at once to capture on different channels in one session, and separate
  them with `frame.interface_name`.

## Requirements

- macOS and Wireshark 4.x. Developed and tested on an Intel MacBook Pro
  (macOS 26) with Wireshark 4.6. Linux has not been tested.
- Python 3 (included with the macOS command line tools).
- `uwifi`, built and working (next section).
- A USB Wi-Fi adapter supported by `uwifi`. Tested through this extcap with
  mt7921u, mt7925u, mt76x2u and rtw8814au adapters. 160MHz capture was verified
  on 6GHz with an mt7925u (receive only).

## 1. Install uwifi

Source: https://github.com/kismetwireless/userspace-wifi
Project page: https://kismetwireless.net/userspace-wifi/

```bash
# dependencies (Homebrew)
brew install libusb pkg-config
brew install zstd xz          # optional: compressed firmware support

git clone https://github.com/kismetwireless/userspace-wifi.git
cd userspace-wifi
make
sudo make install
```

Check that it works before going further:

```bash
uwifi list                    # your adapters should appear
sudo uwifi info               # bands, channels, widths
sudo uwifi capture -c 6 -w - 2>/dev/null | tshark -r - -c 10
```

If `tshark` prints 802.11 frames, you are ready. In a zsh window opened before
the install, run `rehash` so the shell finds the new command.

## 2. Install the extcap

Ask Wireshark where its personal extcap folder is. It varies between installs.

```bash
tshark -G folders | grep -i extcap
```

Use the "Personal Extcap path" line, then:

```bash
git clone https://github.com/gacragg-github/uwifi-wireshark-extcap.git
cd uwifi-wireshark-extcap

mkdir -p "<personal extcap path>"
cp uwifi_extcap "<personal extcap path>/"
chmod +x "<personal extcap path>/uwifi_extcap"
python3 -m py_compile "<personal extcap path>/uwifi_extcap" && echo "syntax ok"
```

Edit `sudoers.example` for your username and the path from `command -v uwifi`,
then install it. Read the Security section first.

```bash
sudo visudo -f /etc/sudoers.d/uwifi
```

Build the channel cache once, then refresh Wireshark:

```bash
"<personal extcap path>/uwifi_extcap" --refresh
```

In Wireshark choose **Capture → Refresh Interfaces**. On a Mac laptop, F5 is Fn+F5.

## 3. Use it

1. Open Wireshark. Your adapters appear on the capture start page as
   "uwifi <name> (<driver> <usb id>, <speed>)".
2. Click the gear icon next to an adapter. Pick a channel from the dropdown, or
   type a channel list in the custom field (it overrides the dropdown).
3. Start the capture. To use several adapters, select them together
   (Cmd-click) and put each on a different channel.
4. The reload button next to the channel list re-reads that adapter's supported
   channels. Use it when no capture is running on the adapter.

### Channel notation

The number is the **primary channel**. The width suffix follows `uwifi`'s parser.

| Example | Meaning |
|---------|---------|
| `6` | 2.4GHz channel 6, 20MHz |
| `36VHT80` | 5GHz, primary 36, 80MHz |
| `36VHT160` | 5GHz, 160MHz (untested) |
| `6g:101` | 6GHz channel 101, 20MHz |
| `6g:101HE160` | 6GHz, primary 101, 160MHz (reports 6455MHz) |
| `6g:97HE160` | 6GHz, primary 97, 160MHz (reports 6435MHz) |
| `1,6,11,36VHT80` | A list to hop through |

6GHz specs need the `6g:` prefix. The dropdown's 160MHz entries use the first
channel of each block as the primary. To keep your AP's own channel as the primary,
use the custom field (for example `6g:101HE160`).

Capture width can affect what you see. In testing, narrower captures showed the
control frames from a wide transmitter but fewer decodable data frames. Match the
width to your AP, and compare adapters on the same traffic at the same time.

Useful display filters:

```
frame.interface_name == "mt7925-0"
wlan.fc.type_subtype != 0x08          # hide beacons
wlan.addr == aa:bb:cc:dd:ee:ff        # follow one device
```

## Files and commands

| Item | Purpose |
|------|---------|
| `uwifi_extcap --refresh` | Rebuild the channel cache for all adapters from a terminal |
| `~/.cache/uwifi_extcap.json` | Cached adapter and channel data |
| `/tmp/uwifi_extcap.log` | Errors from `uwifi`, written during captures and refreshes |

`uwifi` prints a per-channel statistics table (frames, bad FCS, legacy/HT/VHT/HE
counts) to stderr. The extcap saves it in the log.

## Troubleshooting

- **Interfaces do not appear.** Run `uwifi_extcap --extcap-interfaces` and
  `tshark -D`. A Python traceback means the script is broken. Mixed tabs and spaces
  after editing are a common cause, and `python3 -m py_compile` catches them. Confirm
  the script is in the folder reported by `tshark -G folders`.
- **Fewer adapters than expected.** Compare with `uwifi list`. An adapter held by
  another process (a running Kismet, for example) or a USB hub problem can hide it.
- **Dropdown shows only a few channels.** Run `uwifi_extcap --refresh` or press the
  reload button. Both need the sudoers rule.
- **No 160MHz entries.** The adapter does not report `160MHz`. Check with
  `sudo uwifi info -d N | grep Widths`, then reload.
- **A capture starts but shows no packets.** Read `/tmp/uwifi_extcap.log`. Typical
  causes are a sudoers path that does not match `command -v uwifi`, an adapter held
  by another process, or a replugged device.
- **Reload fails during a capture.** `uwifi info` cannot open an adapter that is in
  use. Stop the capture on it first.
- **Adapter names change.** `mt7921-0`, `-1` and `-2` come from enumeration order and
  can shift when devices are replugged. Captures look the adapter up by name each time.
- **Lua errors on startup.** These come from plugins in your personal Wireshark
  plugins folder and are unrelated to this extcap.

## Security

This tool runs `uwifi` as root through `sudo` so Wireshark can start captures
without a password prompt. Read this before installing the sudoers rule.

**What the rule allows.** `NOPASSWD` for `uwifi` lets any process running as your
user run `uwifi` as root with any arguments. `uwifi capture -w <path>` writes files,
and options such as `-F` change which firmware directory root loads from. A
compromised account, or any program you run, could use the rule for root-level file
writes. Treat the rule as giving that user root.

**Protect the binary.** A passwordless rule on a binary that your user can replace is
a direct privilege escalation. On Intel Macs with Homebrew, `/usr/local/bin` is often
owned by your user. Check:

```bash
ls -ld /usr/local/bin
ls -l "$(command -v uwifi)"
```

If the directory is user-writable, install `uwifi` under a root-owned path (every
directory in the path owned by root and not writable by others) and point the
sudoers rule at that copy.

**Reduce the exposure.**

- A small root-owned wrapper that accepts only the arguments this extcap needs, and
  is the only thing allowed in sudoers, would be safer. This repository does not
  ship one yet.
- Do not rely on wildcard argument patterns in sudoers to restrict what `uwifi` can
  do. They match more than they appear to.
- Install the rule only on machines you control, and remove it when done:
  `sudo rm /etc/sudoers.d/uwifi`.
- The log in `/tmp` is readable by other local users.

**Use it lawfully.** Monitor-mode capture can collect other people's traffic and
metadata. Capture only where you are authorized to, and follow local regulations.
`uwifi inject` transmits frames and is not used by this extcap.

## Limitations

- 5GHz 160MHz and 320MHz entries are untested. `uwifi` parses `EHT320`, but none of
  the tested adapters report 320MHz.
- 5GHz 40/80/160MHz and 6GHz 80/160MHz blocks come from standard channel plans and the
  channels the adapter reports, not from per-chip probing. `uwifi info -p` probes what
  actually tunes.
- Hop presets pass the adapter's reported widths to `uwifi capture -W`. Hop mode with
  160MHz is untested.
- Comparisons between adapters so far used sequential captures, not simultaneous ones.

## Credits

Built on userspace-wifi from the Kismet project. This repository contains no
userspace-wifi code.

## License

Choose a license and add a `LICENSE` file.