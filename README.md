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
  on 5GHz and 6GHz with an mt7925u (receive only).

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

Install a sudoers rule so the extcap can run `uwifi` as root without a password
prompt. Read the Security section first. The rule is one line (also in
`sudoers.example`), with your username and the path from `command -v uwifi`:

```
yourusername ALL=(root) NOPASSWD: /usr/local/bin/uwifi
```

Build it, check the syntax with `visudo -c`, then install it:

```bash
echo "$(whoami) ALL=(root) NOPASSWD: $(command -v uwifi)" > /tmp/uwifi.sudoers
cat /tmp/uwifi.sudoers
sudo visudo -cf /tmp/uwifi.sudoers
sudo install -m 440 -o root -g wheel /tmp/uwifi.sudoers /etc/sudoers.d/uwifi
```

Or edit it in place with `sudo visudo -f /etc/sudoers.d/uwifi`, which checks the
syntax when you save.

Confirm it works without a password (`sudo -k` clears any cached credentials, and
`-n` makes `sudo` fail instead of prompting):

```bash
sudo -k
sudo -n uwifi list
```

If `command -v uwifi` is not `/usr/local/bin/uwifi`, also change the `UWIFI` line
near the top of `uwifi_extcap` to match.

**Running without sudo.** On some Macs `uwifi` works without root. Check:

```bash
uwifi info -d 0
uwifi capture -d 0 -c 6 -w - 2>/dev/null | tshark -r - -c 10
```

If both work, set `USE_SUDO = False` near the top of the installed `uwifi_extcap`
and skip the sudoers rule. This is also the way around managed Macs where a
privilege-management tool (for example BeyondTrust/Avecto) adds its own password
prompt to `sudo` and ignores `NOPASSWD`.

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

The dropdown's 160MHz entries are labelled by block, for example
"5GHz ch 36  160MHz (VHT160, 36-64)", and use the first channel of the block as
the primary. To keep your AP's own channel as the primary, use the custom field
(for example `44VHT160` or `6g:101HE160`).

Useful display filters:

```
frame.interface_name == "mt7925-0"
wlan.fc.type_subtype != 0x08          # hide beacons
wlan.addr == aa:bb:cc:dd:ee:ff        # follow one device
```

## 4. Examples

### List adapters

`uwifi list` works without `sudo`. The index is what `-d` takes, and the interface
name (for example `mt7925-0`) is what appears in Wireshark.

```
$ uwifi list
index interface    driver     usb id     usb path     speed
0     mt7921-0     mt7921u    0e8d:7961  0-4.3.1.3    5Gbps
1     mt7925-0     mt7925u    0846:9072  0-4.3.2      5Gbps
2     mt7921-1     mt7921u    0e8d:7961  0-4.3.4      5Gbps
3     mt7921-2     mt7921u    0e8d:7961  0-4.4        5Gbps
4     rtw8814a-0   rtw8814au  0bda:8813  20-4.3.2.1.1.4 480Mbps
```

Indexes can change when adapters are replugged. The extcap looks adapters up by
interface name each time you capture.

### Show what an adapter supports

`uwifi info` loads firmware and needs `sudo`. The `Widths:` lines are what the
extcap uses to build each adapter's dropdown, so an adapter without `160MHz`
there gets no 160MHz entries. The MAC address below is masked.

<details>
<summary><code>sudo uwifi info -d 1</code> (mt7925u)</summary>

```
$ sudo uwifi info -d 1
Opening adapter 1 and loading firmware...
Adapter:   Wireless_Device (0846:9072) on USB 0-4.3.2 at 5Gbps, driver mt7925u
MAC:       00:11:22:33:44:55
Firmware:  ____000000-20260813113118
Antennas:  tx 0x3 rx 0x3

2.4GHz band: 14 channels (14 enabled)
  Channels:
      1      2      3      4      5      6      7      8      9     10     11     12
     13     14
  Legacy rates: 1 2 5.5 11 6 9 12 18 24 36 48 54 Mbps
  HT:        2 streams, MCS 0-15, 40MHz, SGI40, LDPC, RX STBC
  HE:        2 streams, MCS 0-11
  Widths:    20MHz 40MHz

5GHz band: 28 channels (28 enabled)
  Channels:
     36     40     44     48     52     56     60     64    100    104    108    112
    116    120    124    128    132    136    140    144    149    153    157    161
    165    169    173    177
  Legacy rates: 6 9 12 18 24 36 48 54 Mbps
  HT:        2 streams, MCS 0-15, 40MHz, SGI40, LDPC, RX STBC
  VHT:       2 streams, MCS 0-9, 160MHz, SGI80, LDPC
  HE:        2 streams, MCS 0-11
  Widths:    20MHz 40MHz 80MHz 160MHz

6GHz band: 59 channels (59 enabled)
  Channels:
      1      5      9     13     17     21     25     29     33     37     41     45
     49     53     57     61     65     69     73     77     81     85     89     93
     97    101    105    109    113    117    121    125    129    133    137    141
    145    149    153    157    161    165    169    173    177    181    185    189
    193    197    201    205    209    213    217    221    225    229    233
  Legacy rates: 6 9 12 18 24 36 48 54 Mbps
  HE:        2 streams, MCS 0-11
  Widths:    20MHz 40MHz 80MHz 160MHz
```

</details>

To see only the widths on every adapter:

```bash
for i in 0 1 2 3 4; do echo "== $i"; sudo uwifi info -d $i | grep -E 'GHz band|Widths'; done
```

### Capture to the screen with tshark (no Wireshark GUI)

`uwifi capture -w -` writes a radiotap pcap stream to stdout. Its status text goes
to stderr, so discard stderr when piping.

```bash
sudo uwifi capture -d 1 -c 6g:101HE160 -w - 2>/dev/null | tshark -r - -c 10
```

Illustrative output (yours will differ):

```
    1   0.000000  aa:bb:cc:dd:ee:01 → Broadcast    802.11  408 Beacon frame, SN=3486, FN=0, Flags=........, BI=100, SSID="ExampleNet"
    2   0.000301  aa:bb:cc:dd:ee:01 → Broadcast    802.11  402 Beacon frame, SN=3487, FN=0, Flags=........, BI=100, SSID="ExampleNet"
    3   0.037213  11:22:33:44:55:66 → aa:bb:cc:dd:ee:01  802.11  60 Request-to-send, Flags=........C
    4   0.037400  11:22:33:44:55:66 → aa:bb:cc:dd:ee:01  802.11  56 Clear-to-send, Flags=........C
```

Other useful forms:

```bash
# stop after 20 seconds and save a file
sudo uwifi capture -d 1 -c 44VHT160 -w /tmp/ch44.pcap -T 20

# frame-type breakdown of a capture
tshark -r /tmp/ch44.pcap -T fields -e wlan.fc.type_subtype 2>/dev/null | sort | uniq -c | sort -rn
```

macOS's built-in `tcpdump` could not read this stream in testing ("unknown file
format"), while tshark could, so use tshark or Wireshark to read it.

### Choose a channel and width

The number is the primary channel and the suffix sets the width. 6GHz specs need
the `6g:` prefix. A checkmark means the spec was run on real hardware while writing
this README (the 160MHz ones on an mt7925u). The others follow the syntax in
`uwifi capture --help` and the `uwifi` source, but were not individually run.

| Band | Width | `-c` spec | Run |
|------|-------|-----------|-----|
| 2.4GHz | 20MHz | `6` | ✓ |
| 2.4GHz | 40MHz | `1HT40+` or `11HT40-` | |
| 5GHz | 20MHz | `44` | ✓ |
| 5GHz | 40MHz | `36HT40+` | |
| 5GHz | 80MHz | `44VHT80` | ✓ |
| 5GHz | 160MHz | `44VHT160` | ✓ |
| 6GHz | 20MHz | `6g:101` | ✓ |
| 6GHz | 80MHz | `6g:101HE80` | ✓ |
| 6GHz | 160MHz | `6g:101HE160` (primary 101) | ✓ |
| 6GHz | 160MHz | `6g:97HE160` (primary 97) | ✓ |

For example:

```bash
sudo uwifi capture -d 1 -c 6       -w - 2>/dev/null | tshark -r - -c 5   # 2.4GHz ch 6
sudo uwifi capture -d 1 -c 44VHT80  -w - 2>/dev/null | tshark -r - -c 5   # 5GHz 80MHz
sudo uwifi capture -d 1 -c 6g:101HE160 -w - 2>/dev/null | tshark -r - -c 5   # 6GHz 160MHz
```

### Hop across channels

```bash
# a list of channels
sudo uwifi capture -d 0 -c 1,6,11,36VHT80 -T 60 -w /tmp/hop.pcap

# every channel in selected bands at selected widths, 250 ms per channel
sudo uwifi capture -d 1 -b 2,5 -W 20,40,80 -t 250 -T 60 -w /tmp/hop.pcap

# 6GHz at 160MHz only (hopping with 160MHz is untested)
sudo uwifi capture -d 1 -b 6 -W 160 -T 60 -w /tmp/hop160.pcap
```

### Use the extcap from the command line

Check that Wireshark sees the interfaces:

```bash
tshark -D | grep -i uwifi
```

Run the script the way Wireshark does, to see its output:

```bash
"<personal extcap path>/uwifi_extcap" --extcap-interfaces
"<personal extcap path>/uwifi_extcap" --extcap-config --extcap-interface mt7925-0
```

You can also drive a capture by hand through a named pipe. The sudoers rule must
be installed. This method was not run while writing this README.

```bash
mkfifo /tmp/uwifi.fifo
tshark -r /tmp/uwifi.fifo -c 10 &
"<personal extcap path>/uwifi_extcap" --capture --extcap-interface mt7925-0 \
    --fifo /tmp/uwifi.fifo --preset 6g:101HE160
```

### Use the extcap in Wireshark

1. Capture → Refresh Interfaces (Fn+F5 on a Mac laptop).
2. Click the gear next to `uwifi mt7925-0 (...)`.
3. Pick "6GHz ch 97  160MHz (HE160, 97-125)" from the dropdown, or type
   `6g:101HE160` in the custom field to keep channel 101 as the primary.
4. Start. To capture on two adapters at once, select both and give each a different
   channel, then filter with `frame.interface_name == "mt7925-0"`.

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

- 320MHz entries are untested. `uwifi` parses `EHT320`, but none of the tested
  adapters report 320MHz.
- The dropdown offers 40MHz only on 2.4GHz and 5GHz. 6GHz 40MHz is not generated;
  use the custom field if your adapter and `uwifi` support it.
- Hop mode with 160MHz in the width list is untested.
- 5GHz 40/80/160MHz and 6GHz 80/160MHz blocks come from standard channel plans and the
  channels the adapter reports, not from per-chip probing. `uwifi info -p` probes what
  actually tunes.
- 160MHz was verified only on the mt7925u, and only for receive.
- Comparisons between adapters and widths so far used sequential captures, not
  simultaneous ones.

## Disclaimer

This software is provided as is, without warranty of any kind. Use it at your own
risk. The authors are not responsible for any damage, data loss, or legal issue
arising from its use, including running `uwifi` with root privileges and capturing
network traffic. You are responsible for making sure your use is authorized and lawful.

## Credits

Built on userspace-wifi from the Kismet project. This repository contains no
userspace-wifi code.

## License

Licensed under the GNU General Public License, version 2 or (at your option) any
later version (GPL-2.0-or-later), the same license family as Kismet. See
[LICENSE](LICENSE).