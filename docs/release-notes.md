# Unreleased (support 0.5.2)

No new kernel. v0.5.0 stays on `9dbf932` (kernel series 6, support 0.5.1).

## Fingerprint reader

The reader was listed as Unsupported. It is a Goodix 27c6:650a on
`usb_1_ss1` (a800000), the CRD's second USB-C port controller, wired
internally and USB 2.0 only. Three devicetree changes bring it up; no
userspace workaround is needed.

- **Repeater**: `usb_1_ss1_hsphy` pointed at the eUSB2 repeater of an
  smb2360 at SPMI SID 0xa, which this laptop does not have, so the port
  never came up. It now uses the repeater of the smb2360 at SID 0xb, and
  the controller runs on that PHY only (`maximum-speed = "high-speed"`,
  no SuperSpeed port or QMP PHY). With this alone the reader enumerated
  on about half the boots.
- **`vreg_l2d_0p9` always on**: l2d feeds the unused USB3 QMP PHY's PLL.
  Without it the controller sometimes cannot finish its core soft reset
  ("DWC3 controller soft reset failed") and the reader is gone for that
  boot: 3 failures in 12 boots before the TLMM 7 fix, 1 in 20 after it.
  Windows votes LDO2_D for `\_SB.USB1`.
- **TLMM 7 is the reader's reset**: `redriver@4b` (PTN3222 on i2c5, for
  the webcam on `usb_2`) had `reset-gpios = <&tlmm 7 GPIO_ACTIVE_LOW>`,
  taken from the ThinkBook 16. On the 83N3 nothing answers at 0x4b, and
  driving TLMM 7 low drops the reader off the bus; driving it high brings
  it back. `phy_nxp_ptn3222` asserted that reset from its probe until
  `usb_2` initialised its PHY. If that window overlapped the reader's
  enumeration, the reader failed with -71 for the whole boot; if it came
  after, the reader failed its next USB reset, which fprintd does on every
  open. With l2d on and the reset in place: 17 of 20 boots. The node now
  has no reset GPIO and no pin state, and stays so `usb_2_hsphy` keeps
  voting LDO13_B and LDO4_B. The webcam works as before.

Measured on an 83N3 after each reboot: `lsusb`, `fprintd-list`, and the
webcam (`lsusb` and `/dev/video0`). With all three changes, 80 of 80
boots, with no -71 and no soft reset failure. Those 80 include 20 with
the reader left on USB autosuspend (libfprint's hwdb default), where
fprintd woke it every time. The reader also stayed up across a deep
suspend on battery with the lid closed, and fprintd worked after resume.

Earlier attempts at the -71 failures (starting the ADSP after the reader,
force-loading the eUSB2 PHY in the initramfs, keeping `phy_nxp_ptn3222`
out of the initramfs) only moved the timing of the TLMM 7 reset. None of
them is needed with the reset removed: 20 of 20 boots without each.

## Mute and volume ignored the Hyprland session

Support 0.5.1 opens the keyboard, then logged `no active seat` on every
key. `loginctl list-sessions` on this laptop has session 1 as class
`manager` with no seat, and session 2 as the Hyprland session on
`seat0` (`ActiveSession=2`, uid 1000, class user). The seat file's
`ACTIVE=` field is that session id, not a boolean, so a check for
`ACTIVE=1` skipped session 2. Support 0.5.2 asks `loginctl show-seat
seat0 -p ActiveSession` and uses that session's `User` when `Class=user`
and `Active=yes`. `pactl` still runs as that uid with
`XDG_RUNTIME_DIR=/run/user/<uid>`. Not confirmed on hardware yet.

## The fan curve write was acknowledged and the fan stayed off

Support 0.5.1 logged `programmed fan 1 curve for profiles 1-7`. Under an
8-core load the hottest zone was `cpu1-3-btm` at 73.8 °C and fan 1 was
still `0x02 0x00 0x00` (0 rpm). An I2C ACK is not proof the EC stored
the curve. Support 0.5.2 reads slot `0x61` back after the write and logs
the bytes. `debondragon-fan probe` reads fan 1 slots for profiles 1–7,
plus slot `0x61` with a zero second byte. It still does not write the
profile, debug mode, or EC enable. Qualcomm's Hamoa driver has no
separate apply-LUT command: it writes the profile register and the LUT,
and its only other init writes are SoC Tj, modern-standby, and SCI
enable (`0x35` = 1). `0x35` = 0 is "SCI EC Disable" and is not written.
The fan is still Experimental.

---

# DebOnDragon v0.5.0

Kernel `7.2.0-debondragon-6` (Linux v7.2) and `debondragon-support` 0.5.1.
Series 6 installs beside series 5. v0.4.0 stays on series 5 and support 0.4.1.

The support package is 0.5.1. The 0.5.0 package was already installed from
the untagged series-6 image, and a deb with the same version would not
upgrade it. There is no new kernel in 0.5.1.

## The fan was not seeing the SoC

On series 5, light use left every SoC thermal zone at 59–63 °C while the
battery read 30 °C and the PMICs 37 °C. The zones matching each other is
a fan that is off or barely spinning. Windows feeds die temperature to
the EC ("Temp to EC", register 0x20). Linux never did, and the CPU zones
had no passive trip short of a 115 °C shutdown.

Kernel `7.2.0-debondragon-6` adds a 75 °C passive trip on the CPU and
CPUSS zones (cluster 0 cools cpu0–3, cluster 1 cools cpu4–7) and lowers
the GPU passive trip from 95 °C to 85 °C. The 115 °C critical trips stay.
Nothing in the devicetree writes the EC.

`debondragon-support` 0.5.0 adds `debondragon-fan`. It is off until
`sudo debondragon-fan enable`. It reports SoC temperature (register 0x20,
decidegrees Celsius, sensors 0x01/0x02/0x03) and tells the EC about
suspend (register 0x23). It does not set RPM, replace the fan curve, or
write debug mode. `qcom-hamoa-ec` stays blacklisted: on this EC that
driver's reads are the wrong protocol and its probe would write debug
mode. Steps and the power-off warning are in
docs/hardware-testing.md, "Fan and thermals".

## The fan stayed off at 72.7 °C

On series 6 with support 0.5.0 the reporter was running and the writes
were the decidegree `0x20` reports above. Idle SoC zones were 48–50 °C.
After 90 seconds of all-core load the hottest zone was `cpu0-0-btm` at
72.7 °C. Fan 1 answered `0x02 0x00 0x00` (0 rpm) and fan 2 answered
`0x00 0x00 0x00`. The fan was physically stopped: no sound, no airflow.
The 0 rpm decode is right. 72.7 °C is already above the DSDT active trip
for the SoC fan zone (0x0D3E, about 66 °C).

A later light-load read, with the reporter still running, gave the same
shape. Hottest zone 55.9 °C, battery 30.3 °C. Fan 1 status `0x00` and
speed `0x02 0x00 0x00` (byte count 2, 0 rpm). Fan 2 speed
`0x00 0x00 0x00` (byte count 0: no second fan). EC thermistors
`0x29`–`0x2d` all `0x00`. Profile `0x24` was `0x06`, which is the
best-performance-on-AC profile, not a fan-off mode. The curve reads used
LUT ids `0x00`, `0x10` and `0x01`. The id the DSDT and Qualcomm's driver
use for fan 1 profile 6 is `0x61` (`profile << 4 | fan`), so those buffers
were not the active curve. One of them trailed into `0x03 0x00 0x18 0x01`,
the EC firmware version from register `0x0e`.

Qualcomm's Hamoa driver sends sensor `0x01` only, in 0.1 °C units
(651 means 65.1 °C), and it programs the fan LUT itself. The unit we were
already sending matches that. Sending tenths of Kelvin for 72.7 °C would
look like 345.9 °C, so the unit is not changed. What was missing is the
curve: temperature reports with no LUT left the fan at 0 rpm.

Support 0.5.1, once the reporter is enabled, writes that curve for fan 1
and profiles 1–7. The table is the Hamoa breakpoints. Speed is off below
52 °C, 1600 rpm at 55 °C, 2800 rpm at 70 °C, and never above 4000 rpm.
The write is register `0x28`, 68 bytes, temp source `0x01`. It does not
change the selected profile, set a direct RPM, or write debug mode
(`0x30`) or EC enable (`0x35`). `qcom-hamoa-ec` stays blacklisted. The
temperature report is now sensor `0x01` only. `debondragon-fan probe`
stays read-only and reads the `0x61` slot. Not confirmed on hardware:
the fan is still Experimental until it is heard to spin.

## Mute and volume keys could not open the keyboard

Support 0.4.1 started `debondragon-volkeys` in the user session. On the
83N3 the keys arrive on `platform-a84000.i2c-event-kbd` (`KEY_MUTE`,
`KEY_VOLUMEDOWN`, `KEY_VOLUMEUP`). That node is mode 660, `root:input`.
The session user is not in group `input`, so the open failed, and the
program logged nothing.

Support 0.5.1 runs it as a system service. It opens the node as root,
ignores every key except mute and volume, and runs `pactl` as the
active seat user. If it cannot open a device it logs that, and the
message says not to add the user to group `input`. A udev rule grants
`uaccess` only to a consumer-control node on `a84000.i2c` that is not
the typing keyboard. The 0.4.1 user service is masked and its autostart
entry is removed. Not confirmed on hardware. Steps:
docs/hardware-testing.md, "Mute and volume keys".

---

# DebOnDragon v0.4.0

Kernel `7.2.0-debondragon-5` (Linux v7.2) and `debondragon-support` 0.4.1.
Series 5 is the default. It installs beside earlier kernels
(docs/kernel-upgrade.md, "Upgrade to v0.4.0").

## Audio works

Verified on the 83N3 with series 5 and support 0.4.0: the sound card and
the HiFi profile come up, PulseAudio lists Speaker and Headphones, and
`speaker-test` plays Front Left. Front Right is silent on this unit in
Debian and in Windows, so that speaker is a hardware fault. Headset and
internal microphones are not confirmed yet. HDMI audio stays silent (the
topology has no DisplayPort graphs). HDMI video stays on.

The PA volume is capped at 0 dB. There is no active speaker protection.
Keep the volume moderate.

## After boot, PulseAudio stayed on auto_null

PulseAudio starts at login. The sound card appears only once the ADSP
firmware has loaded, which is later, and PulseAudio then kept `auto_null`
until `systemctl --user restart pulseaudio`. Support 0.4.1 restarts it
once a playback PCM shows up, and only if the sinks are still `auto_null`.

## debondragon-firmware

`debondragon-firmware` with no arguments failed with "No Windows
DriverStore or firmware folder found" when the Windows NTFS partition was
already mounted at `/mnt/windows` (mounting it again fails, and the
existing mount was ignored). It now uses an NTFS filesystem that is
already mounted. `--help` prints the usage; it used to be treated as a
folder path. Passing the FileRepository path still works.

The device-signed ADSP firmware is still not in the image. Copy it with
`debondragon-firmware` or `windows/Export-Firmware.ps1`.

## Mute and volume keys

F1, F2 and F3 (mute, volume down, volume up) did nothing. Support 0.4.1
runs `debondragon-volkeys` in the user session and calls `pactl` for
`KEY_MUTE`, `KEY_VOLUMEDOWN` and `KEY_VOLUMEUP`. Hardware later showed
those events on the typing-keyboard node
(`platform-a84000.i2c-event-kbd`), which that user service cannot open,
and the failure was silent. The fix is in support 0.5.1 (above). v0.4.0
itself is unchanged.

## Why series 5 had to describe the speakers

Support 0.2.1 already made PulseAudio show a Headphones sink. Nothing
played until the default devicetree enabled the two WSA8845 on SoundWire
0 and the UCM profile stopped using the Debian sequences that this kernel
rejects (PA volume 12, digital volume 84, class-H headphones). Details
are in the board notes. Series 4 remains a rollback: its devicetree does
not wait for the speaker amplifiers.

---

## HDMI video (kernel series 4, support 0.3.0)

Kernel `7.2.0-debondragon-4` (Linux v7.2) and `debondragon-support` 0.3.0.
Installs beside earlier kernels; don't reinstall Debian
(docs/kernel-upgrade.md, "Upgrade from v0.3.0").

## HDMI works

Verified on the 83N3 with a Samsung LS27D300G: 1920x1080@60 over a
4-lane HBR2 DisplayPort link to the HDMI converter (Parade PS176), extended
desktop under Hyprland with pointer and windows moving between screens.

The fix is one devicetree line: the DP lane orientation of the PHY that
feeds the converter (`data-lanes = <3 2 1 0>`; the earlier `<0 1 2 3>`
made link training fail). Kernel -4 is otherwise identical to -3.

`debondragon-support` 0.3.0 turns HDMI on automatically
(`DEBONDRAGON_HDMI=auto`) for kernels with this fix and keeps it off for
older ones; GRUB's "DebOnDragon kernels" submenu has a "HDMI off" entry
for -4 as a fallback. No sysfs commands needed.

**Not yet verified:** HDMI at boot and hotplug with the new automatic
setting, long runs, reboot and unplug/replug with active video (earlier
kernels reset the laptop on unplug after failed training). Tests:
docs/hdmi-testing.md.

## Bluetooth headsets: microphone and Earbuds app

- `debondragon-support` 0.3.0 makes PulseAudio register the Bluetooth
  headset profiles itself (`/etc/pulse/default.pa.d/debondragon-bluetooth.pa`,
  native backend). Before, headsets could only play music; their
  microphone was unusable. It also reports headset battery levels to BlueZ.
- `apps/earbuds/`: a GTK 4 + libadwaita app (pair, connect, music vs.
  call mode, volume, microphone level, battery) and a Waybar module; install
  with `apps/earbuds/install.sh`. Noise-control switching for realme earbuds
  isn't possible yet (undocumented protocol). Not yet tested with real
  earbuds.

Unchanged and still working: touchpad (TLMM 19), internal display, GPU,
brightness, camera, Wi-Fi, keyboard, touchscreen, USB-A.

---

# DebOnDragon v0.3.0

For the Lenovo IdeaPad Slim 3 15Q8X10 (83N3, Snapdragon X X1-26-100).
Kernel series `7.2.0-debondragon-3` (Linux v7.2) and `debondragon-support` 0.2.1.

**Installed system: don't reinstall.** The new kernel installs **beside**
7.2.0-debondragon-2. Test it once without changing the default:
https://github.com/SatoruGoj0/DebianOnDragon/blob/main/docs/kernel-upgrade.md
("Test a new kernel once, keeping the current one as default").

## Verified on hardware

- **Touchpad works** (kernel `7.2.0-debondragon-3`): its interrupt is on
  TLMM 19, decoded from the DSDT, not the CRD's TLMM 3. Interrupts arrive,
  libinput reports continuous pointer motion, normal use is fine.
- Kernel -3 boots; HDMI stays off (`video=HDMI-A-1:d`).
- Audio: the UCM profile now loads and PulseAudio offers a Headphones sink
  (was `auto_null`). Sound output itself is **not tested yet**.
- Confirmed on v0.2.0 and unchanged here: internal display and GPU
  acceleration, brightness control, camera, Wi-Fi, keyboard, touchscreen,
  USB-A, temperature sensors (on -3: display up, camera enumerates).

Still broken: HDMI (link training fails; unplugging used to reset the
laptop, hence off by default). The EC/fan driver's protocol doesn't match
this EC (driver stays off; the EC runs the fan itself).

## Changes

`debondragon-support` 0.2.1 (works with kernel -2 as well):
- **HDMI off by default** (`video=HDMI-A-1:d`), so the failing link is
  never enabled and there is nothing to tear down. Reversible:
  `DEBONDRAGON_HDMI=on` in `/etc/default/debondragon`, only to test a fix.
- **Brightness**: saved once a minute when it changed and restored exactly
  at boot; without a saved level, at least `DEBONDRAGON_MIN_BRIGHTNESS`
  (30%). Restore across a reboot not yet confirmed.
- **Audio profile** under the card's long name, which is what ALSA looks up
  (the short name is cut to 31 characters). Speakers stay off.
- **`DEBONDRAGON_DEFAULT_KERNEL`** pins GRUB's default to one kernel with
  its own devicetree, so a new kernel can be tried once with `grub-reboot`.

Kernel `7.2.0-debondragon-3`:
- **Touchpad interrupt on TLMM 19** (was the CRD's TLMM 3, which never
  fired), level-low with pull-up, decoded from the DSDT
  (`docs/board-notes.md`). **Verified on hardware.**
- **Absent PMIC (SPMI SID 0xa) disabled**: stops a failing probe and its
  boot WARNING. No working device used it.
- **`/dev/drm_dp_auxN`** (read-only HDMI diagnostics) and **lockup / hung
  task detectors** (report only, never reboot). HDMI link training is
  **not** fixed yet.
- USB tethering drivers (RNDIS, ipheth).

EC/fan driver: still blacklisted.

---

Previous release (v0.2.0):

DebOnDragon for the Lenovo IdeaPad Slim 3 15Q8X10 (83N3, Snapdragon X X1-26-100).
Linux v7.2 kernel series `7.2.0-debondragon-2`.

**Installed system:** don't reinstall. Download `linux-image-7.2.0-debondragon-2_*.deb`,
`debondragon-support_*_all.deb` and `SHA256SUMS`, then follow
https://github.com/SatoruGoj0/DebianOnDragon/blob/main/docs/kernel-upgrade.md
The new kernel installs **next to** the old one. GRUB's "DebOnDragon kernels"
submenu boots either one with its own devicetree.

New in this kernel (implemented from hardware evidence; **not yet tested on hardware**):
- Touchpad: interrupt pull-up fix for the `incomplete report` floods
- Display brightness: PWM moved to pmk8550 GPIO 5
- HDMI: DP2 → DP-to-HDMI bridge
- Internal camera: usb_2 + eUSB2 repeater
- Audio: headphones, headset mic and internal mics (DSP topology and UCM come
  with debondragon-support; needs `debondragon-firmware`)
- Internal speakers: **opt-in** `-speakers` devicetree (no speaker protection on Linux)
- Fan/EC driver: built but **off by default**; check with `sudo debondragon-ec-check`

Test steps and the support matrix:
https://github.com/SatoruGoj0/DebianOnDragon/blob/main/docs/hardware-testing.md

**New install:** write the ISO in DD mode, boot it from the USB-A port,
and keep Secure Boot off (https://github.com/SatoruGoj0/DebianOnDragon/blob/main/docs/install.md).
