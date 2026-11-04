# Jomaa KB96 trackpad on Linux

The Jomaa KB96 "Function Trackpad" identifies itself as an Apple Magic Trackpad 2 when it's connected through its USB dongle. Linux then loads the Apple module, `hid-magicmouse`, and the cursor doesn't move. Once that module is blocked, the trackpad works as a normal multitouch touchpad with gestures. You don't need a driver for this, only one line of config.

Tested on CachyOS with kernel 7.3, libinput 1.32 and KDE Plasma 6.8 on Wayland. Other distros are untested.

## Check that this is your problem

Plug in the dongle and run:

```sh
lsusb -d 05ac:0265
lsmod | grep hid_magicmouse
```

If both commands print a line, this fix applies.

## What works

| Connection | Status |
|---|---|
| USB dongle, connection switch set to dongle | Works: cursor and 4-finger gestures in KDE |
| Bluetooth | Mouse mode only, no touchpad gestures |

Not tested yet: whether the fix survives a reboot or replugging. Two-finger scrolling and clicking haven't been checked one by one either.

## Side effect on Apple devices

The config blocks the `hid_magicmouse` module for the whole machine, so a real Apple Magic Mouse or Magic Trackpad won't work properly while it's installed.

## Install

```sh
git clone https://github.com/HanreichC/jomaa.kb96trackpad.driver.git
cd jomaa.kb96trackpad.driver
sudo install -m 644 function-trackpad.conf /etc/modprobe.d/
sudo rmmod hid_magicmouse   # only if it is currently loaded
```

Then unplug the dongle and plug it back in.

To check the result, run:

```sh
grep -H . /sys/class/input/event*/device/name | grep '05ac:0265 Touchpad'
```

It should print at least one line. If the cursor doesn't move after a reboot, try `sudo mkinitcpio -P` on Arch-based distros. This step is untested.

To uninstall, run `sudo rm /etc/modprobe.d/function-trackpad.conf` and reboot.

## Gestures in KDE Plasma 6.8

| Gesture | Result |
|---|---|
| 4 fingers up | Overview, then the grid on a second swipe (tested) |
| 4 fingers down | Grid (tested) |
| 4 fingers left or right | Recognised by libinput; no visible effect with only one desktop |
| 3 fingers in any direction | Recognised by libinput; no KDE action seen |

Swipe far enough, because KWin cancels a gesture that ends too early.

## Why the cursor doesn't move

Through the USB dongle the trackpad reports `05ac:0265`, the ID of an Apple Magic Trackpad 2, but its HID descriptor describes a standard Precision Touchpad with 5 contacts. The ID makes the kernel load `hid-magicmouse`, and the cursor doesn't move. With that module blocked, `hid-multitouch` takes over, and libinput and KWin see a normal touchpad.

## Why Bluetooth stays in mouse mode

Over Bluetooth the trackpad reports `045e:0040` and declares only a keyboard, a mouse and consumer keys. The firmware recognises gestures itself and sends the result:

| Gesture | What the computer receives |
|---|---|
| 1 finger | Mouse movement |
| 2 fingers | Coarse wheel steps |
| 3 fingers | Left button held while moving |
| 4 fingers | `Meta+Tab` or `Meta+Shift+Tab` |

The device also has touch-style reports (`04`, plus features `02`, `03`, `05` and `06`) that it doesn't declare. Report `04` stayed silent in every test, and writes to features `03` and `05` failed with an I/O error. Patching the descriptor can't help, because no finger data arrives.

Don't write to the vendor service `f000ffc0-...` while experimenting. It looks like TI's over-the-air firmware update service.

## Help with testing

Please open an issue if you've tried this on another distro or kernel, or checked anything listed as untested above.

A udev rule that rebinds only this device would remove the side effect on Apple devices. The repo doesn't have one yet. The KB96 reports no USB manufacturer string, which a rule could match on.
