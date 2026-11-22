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
   "uwifi <name> (<driver> <usb id>,## License

Licensed under the GNU General Public License, version 2 or (at your option) any
later version (GPL-2.0-or-later), the same license family as Kismet. See
[LICENSE](LICENSE).
