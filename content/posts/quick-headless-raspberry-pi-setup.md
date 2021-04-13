+++
title = "Quick Headless Raspberry Pi Setup"
date = 2017-05-10T00:00:00+08:00
draft = false
author = "Moritz"
+++

To run any software on your [Raspberry Pi](http://amzn.to/2qcKh6m), it
clearly needs to be set up. However, without an HDMI capable device or a
USB keyboard we won't get anywhere without a headless setup. You will
need a DHCP capable router, a [Raspberry Pi](http://amzn.to/2qcKh6m),
a [SD card](http://amzn.to/2qcTxY4), and a
[SD card reader/writer](http://amzn.to/2qO1X5f) (if you computer
doesn't already provide one).

We'll use [Raspbian
jessie lite](https://www.raspberrypi.org/downloads/raspbian/) and [etcher](https://etcher.io/). Putting Raspbian onto
the SD card should present no issues. Simply download etcher and
Raspbian, and follow the etcher instructions.

Recent versions of Raspbian have disabled access via SSH by default, in
[an
attempt to prevent the proliferation of Raspberry Pis joining botnets](https://www.raspberrypi.org/blog/a-security-update-for-raspbian-pixel/).

> SSH disabled by default; can be enabled by creating a file with name
> “ssh” in boot partition

_Source:_
[_Raspbian
release notes 2016--11--25_](http://downloads.raspberrypi.org/raspbian/release%5Fnotes.txt)

Thus you will need `/boot/ssh`; simply touching that file should be
sufficient.

```text
touch /Volumes/boot/ssh
```

should do on macOS. _Note: if you don't see_ `/Volumnes/boot` _try
re-inserting the SD card._

Eject the SD card, and boot up your Raspberry Pi.

The Raspberry Pi should come up as `raspberrypi` on your network,
depending on your setup, logging in via SSH might be as simple as

```text
ssh pi@raspberrypi
```

Or you might have to
[figure
out the Raspberry Pis ip address](https://www.raspberrypi.org/documentation/remote-access/ssh/unix.md) from your router. The default
password is **raspberry** and you are strongly advised to change it upon
logging in for the first time using `passwd`.

You can now configure your Raspberry Pi to your liking using

```text
sudo raspi-config
```

I would also suggest to install any pending updates:

```text
sudo aptitude update
sudo aptitude upgrade
```

With all that in place you can enjoy playing with your Raspberry Pi.
Next we will create a cross compilation toolchain and cross compile
software to run on the Raspberry Pi.

--- <br />
If you also have a tontec mz61581 tft display, you might want to add

```text
dtparam=spi=on
dtoverlay=mz61581
```

to your `/boot/config.txt` and append `fbcon=map:10` to your
`/boot/cmdline.txt` to enable the the display during the boot process.

You might also need to download the latest
[mz61581-overlay.dbt](https://www.itontec.com/mz61581-overlay.dtb)
from [itontec](https://www.itontec.com) and place it into
`/boot/overlays`.