# Orion SC008HA Hacks
Chronicling my attempts at hacking the SC008HA IP camera - a Tuya-based IP camera branded Orion here in Australia, by Bunnings (our major hardware chain).

## Contents

- [First look](#first-look)
- [Locating UART](#attempting-to-find-uart)
- [Attempting to gain access](#attempting-to-gain-access)
- [Success! (well, nearly...)](#first-success)
- [Great success - high five!](#total-success)

## Introduction

I was recently given an Orion SC008HA wiress IP surveillance camera by a friend who had no need for it. Knowing I was into home automation and enjoyed meddling with all things technology, he gave it to me in exchange for some home-made pasta sauces, a handful of Cat 6A ethernet cables, and a motion sensor for the light at his front door. Assuming I can get this working as I want, I consider that a good deal.

This repo simply documents my efforts to attempt to hack the camera so I can use it as a plain old RTSP camera wth my Home Assistant server, and not be tied to the application required to use it out of the box. I'm inspired, in part at least, by the efforts of @ant-thomas, who managed to (successfully) hack a ZS-GX1 camera [here](https://github.com/ant-thomas/zsgx1hacks).

## First look

### 2020-07-09

* Photos of the separated "head" and board can be found [here](./photos)

Externally, this camera mostly resembles the ZS-GX1, pictured [here](./img/zs-gx1.jpg), with the only notable exception being the lack of an ethernet socket on the back. Instead, there is only a single micro-USB socket for power.

Internally, there is no mainboard in the base of the camera. The only items of note in the base at the power cables from the USB socket and the 5V DC motor used for panning the camera.

The "head" of the camera separates with very little effort. With the camera lens pointing at you, gently pull the right side of the "head" casing away, being careful not to pull too far - there is a wireless antenna cable attached to the half that you're removing. Once the two halves are separated, you can use a fine blade screwdrive to gently prise off the antenna cable, and put the now separate casing to one side.

The board and lens bezel can now be removed from the remaining half of the "head" casing. With the wireless chip at the bottom left, we can see the following (clock face references used):

* Realteak RTL8188FTV wireless chip (6 o'clock)
* HiSilicon H13518ERNCV300 camera SoC (12 o'clock)
* 2-wire connector for speaker (1 o'clock)
* 4-wire connector for pan motor (3 o'clock)
* 4-wire connector for tilt motor (4 o'clock)
* 2-wire connector for USB power (5 o'clock)

No GPIO pins, or any other interface pads, appear to be available.

## Attempting to find UART

### 2020-10-01

So I left this on a corner of my workbench for a couple of months or so, then saw a post by @koutto on Reddit, linking to [this](https://github.com/koutto/hardware-hacking) brilliant starter tutorial, on hacking hardware devices.

(Re)inspired, I took a closer look at the camera board, and (using my previous clockface reference) I saw a row of 4 pads at 11 o'clock.  Time to take a closer look...

* Pad 1 (left): using the ground pin on the USB power connector, this pad is GND
* Pad 2: testing with GND pad, this appears to have a constant voltage of ~3.4V
* Pad 3: testing with GND pad, this appears to have a constant voltage of ~3.4V, slightly less than pad 2
* Pad 4: testing with GND pad, this appears to have a low, fluctuating voltage, between ~200mV and ~1.1V

So far, my first thoughts are that these are UART pins:

* Pin 1 = GND
* Pin 2 = 3.3V Vcc
* Pin 3 = likely Tx
* Pin 4 = likely Rx

Time to solder some header pins and hook it up to either a Raspberry Pi or an Arduino board, and see what I get.

Success ! (of sorts) - looks like I had Tx and Rx the wrong way around. Plugging them in to the UART pins on a Raspberry Pi and using screen to monitor the serial interface gives up:
```
hisi-sdhci: 0
PPS:Jul 22 2019 00:22:28 meari_c5    0 


please input password::
```

So, to recap, the pins are actually:

* Pin 1 = GND
* Pin 2 = 3.3V Vcc
* Pin 3 = Rx
* Pin 4 = Tx

Now to try brute forcing the password, which will likely be a nuisance, as the prompt  locks up after what looks like a 1 second timeout, unless you hit enter to get the password prompt.  More to come...

## Attempting to gain access

### 2020-10-01

After trying to brute force using a few lists I found around the place, I started researching what detail I could about the device, as there were very few markings on the outside.

Searching around for the prompt (particularly the "meari_c5" part) eventually led me to discover the camera is made by Hangzhou Meari. Even better, I found [this](https://fccid.io/2AG7CMINI8C) FCC ID, with internal photos depicting a board that looks similar to the board in my camera.

My best guess at this stage is that the Orion SC008HA is a new variant of the Meari Mini 8C. Just in case, I grabbed the user manual and internal photos PDFs (you can find them [here](./fcc/)).

Also, when booting up with the reset button pressed, the output is this:

```
hisi-sdhci: 0
PPS:Jul 22 2019 00:22:28 meari_c5    0 
button
cmd:fatload mmc 0 0x42000000 ppsMmcTool.txt 1020
```

Maybe a clue as to something I can do with a file called ppsMmcTool.txt on the microSD card? More searching to do.

### 2020-10-03

So my searching eventually led me to [this](https://github.com/AMoo-Miki/homebridge-tuya-lan/issues/4) thread for Tuya homebridges.

I'll save you the effort of reading about all the passwords I tried but, I did manage to get access to the webserver running on port 80, using the below credentials that I found somewhere in the thread:

	user: admin
	password: 056565099
	
Visiting http://\<ip>/devices/deviceinfo (also found in the thread) yielded some JSON:

```
{
  "devname": "Smart Home Camera",
  "model": "Speed 5X",
  "serialno": "058023258",
  "softwareversion": "2.9.0",
  "hardwareversion": "S5X_H1_V10_F23",
  "firmwareversion": "ppstrong-c5-tuya2_arlec008-2.9.0.20190808",
  "authkey": "kX5WMUgvzRGagO35afpB1B2ySAX0AR6s",
  "deviceid": "pp0188d5a405c78fd88b",
  "pid": "aaa",
  "WiFi MAC": "74:ee:2a:69:22:e4"
}
```

Yes, I left a number of device-specific attributes intact.  I'm fine with it.  If I successfully hack this thing, it'll be a dumb RTSP camera on my IoT VLAN anyway without any internet access.

Anyway, it seems I'm another step closer.  More recon of this webserver is needed.

Further reading of the thread has me thinking this camera is very closely related, if not the same as, a similar camera sold at Walmart in the states, under the Merkury brand.  Another comment in the thread shows that visiting the URL http://\<ip>/search yields:

```
{
	"deviceName":	"058023258",
	"serialno":	"058023258",
	"sn":	"pp0188d5a405c78fd88b",
	"licenseUsed":	1,
	"licenseId":	"pp0188d5a405c78fd88b",
	"p2p_uuid":	"v2-0580232580000111A",
	"factory_code":	0,
	"factory_code_str":	"",
	"model":	"Speed 5X",
	"ip":	"192.168.40.115",
	"mask":	"255.255.255.0",
	"gw":	"192.168.40.1",
	"mac":	"74:ee:2a:69:22:e4",
	"interface":	"wlan0",
	"version":	"2.9.0"
}
```

Also, visiting http://\<ip\>/flash/upgrade/release_package takes me to a screen to upload new firmware:

![](./img/firmware_upload.png)

A bunch of other URLs from this thread also work - here's the full list so far that seem to respond with something, rather than timing out or the browser getting a connection reset:

```
/devices/deviceinfo
/devices/reboot
/devices/temp_humidity/value

/flash/encryption
/flash/identity
/flash/upgrade/all
/flash/upgrade/ppstrong
/flash/upgrade/release_package
/flash/search

/proc/<anything_useful>
```

That last one is interesting - it seems to display the contents of just about anything you'd expect from /proc, eg. /proc/mounts gives:

```
rootfs / rootfs rw,size=15864k,nr_inodes=3966 0 0
proc /proc proc rw,relatime 0 0
sysfs /sys sysfs rw,relatime 0 0
tmpfs /dev tmpfs rw,relatime 0 0
devpts /dev/pts devpts rw,relatime,mode=600,ptmxmode=000 0 0
/dev/mtdblock6 /home/cfg jffs2 rw,relatime 0 0
```

/proc/devices gives:

```
Character devices:
  1 mem
  4 /dev/vc/0
  4 tty
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  7 vcs
 10 misc
 13 input
 29 fb
 89 i2c
 90 mtd
128 ptm
136 pts
180 usb
189 usb_device
204 ttyAMA
218 himedia
253 motor
254 Strnio

Block devices:
259 blkext
 31 mtdblock
179 mmc
```

Checking out /proc/net/tcp gives:

```
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   0: 00000000:1A0C 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 1247 1 c0f91080 100 0 0 10 0                              
   1: 00000000:0050 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 1129 1 c0f90000 100 0 0 10 0                              
   2: 7328A8C0:0050 7700A8C0:DD08 01 00000000:00000000 00:00000000 00000000     0        0 4415 1 c0f92100 23 4 32 10 -1                             
   3: 7328A8C0:C9D6 5A1FB812:22B3 01 00000000:00000000 02:00000BBB 00000000     0        0 1235 2 c0f90580 46 8 26 10 -1                             
   4: 7328A8C0:0050 7700A8C0:DD01 06 00000000:00000000 03:000003EB 00000000     0        0 0 3 c0c80260                                              
```

Which translates to ports 80, 6668 and 51670.  Obviously, 80 makes sense, and 6668 is a known port for Tuya devices, but 51670 doesn't do much, and nmap tells me it's closed.

Looking at /proc/net/udp gives:

```
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops             
  132: 0100007F:E027 0100007F:E28C 01 00000000:00000000 00:00000000 00000000     0        0 1128 2 c2292f00 0                  
  201: 00000000:7D6C 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 1351 2 c2293900 0                  
  211: 00000000:0E76 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 1110 2 c2292780 0                  
  212: 7328A8C0:E177 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 1266 2 c2293680 0                  
  212: 00000000:0E77 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 1111 2 c2292a00 0                  
  233: 0100007F:E28C 0100007F:E027 01 00000000:00000000 00:00000000 00000000     0        0 1127 2 c2292c80 0                  
  255: 00000000:2FA2 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 1350 2 c2293400 0                  
```

These translate to:

1. 57383
2. 32108
3. 3702
4. 57719
5. 3203
6. 57996
7. 12194

I'll keep sniffing around.  The Github thread I linked above is quite active, as of a couple of days ago, which is potentially promising.  I'll definitely be keeping an eye on it.

### 2020-10-04

So I thought I'd see if I could make any progress with the ppsMmcTool.txt file mentioned when booting while holding the reset button.

I touched a file of the same name on a FAT32-formatted microSD card, and got the following when booting:

```
hisi-sdhci: 0 (SD)
PPS:Jul 22 2019 00:22:28 meari_c5    0 
button
cmd:fatload mmc 0 0x42000000 ppsMmcTool.txt 1020
had init
reading ppsMmcTool.txt
** ppsMmcTool.txt shorter than offset + len **
18 bytes read in 3 ms (5.9 KiB/s)
magic err
```

Not sure what this means yet, except that it's possibly expecting the file to be 1020 bytes in length?  I filled out the file with hex 00s until it reached 1020 bytes, but got the same message:

```
hisi-sdhci: 0 (SD)
PPS:Jul 22 2019 00:22:28 meari_c5    0 
button
cmd:fatload mmc 0 0x42000000 ppsMmcTool.txt 1020
had init
reading ppsMmcTool.txt
** ppsMmcTool.txt shorter than offset + len **
1020 bytes read in 3 ms (332 KiB/s)
```

## First success

So it's been quite a while since I came back to this project, during which time a few people came across this repo and reached out to me to discuss my progress and offer their own experiences from trying to hack similar devices.

Unfortunately, this came at an incredibly busy and stressful period for me, culminating in me quitting my job in December (don't worry - I got an awesome new one before I did) and escaping for a week away with the family over Christmas/New Year.

So, finally sitting down a few days ago (late January) to pick this up again, I noticed the incredibly talented [@guino](https://github.com/guino) had opened, and later closed, [issue #1](https://github.com/DanTLehman/orion_sc008ha/issues/1) on this repo.

@guino (real name Wagner) had not only managed to get through the bootloader on this device, but developed a shell hack utilising the ppsMmcTool.txt file that delivered all sorts of wonderful goodies for people to play with and modify.

In other words, Wagner has developed a software hack that only requires a microSD card, and nothing more.

Not only that, but he has also figured out how to reverse engineer and patch the core application on these devices, enabling the Holy Grail of rtsp for those that want it!

It turns out my particular device is running an older version (v2.9.0) of this core app, but I've managed to get mjpeg working.  Patching for rtsp has not yet delivered results for me, but Wagner is very kindly working with me to try and figure out these issues.

So, in short, if you have one of these devices and haven't yet managed to enable mjpeg or rtsp support, please take a look at Wagner's excellent repo on the subject, and specifically read [the issue](https://github.com/guino/BazzDoorbell/issues/2) he opened, describing the process.

If you do have success with this, please consider [shouting Wagner a coffee](http://paypal.me/wbbo) for all his hard work.

I'll keep posting my progress towards rtsp on this device, as it happens.

## Total success

So, after another couple of days, using Wagner's hard work and exchanging lots of emails with him, I finally got the patched version of the core app working with rtsp streaming.  Rather than reinvent most of Wagner's wheel, I'll simply point to the steps I took in his guide, and indicate where I varied any of them for myself.

### A quick recap...

First, it helps to understand what Wagner's hard work gave us, which is a [software-based approach](https://github.com/guino/BazzDoorbell/issues/2) to executing our own arbitrary code at boot-time, enabling us to do things like:

- Turning on a telnet daemon, so we can remotely login to a shell
- Configuring and launching a http server for mjpeg streaming
- Capturing copies of the core app (indeed, anything else on the camera's filesystems) for later analysis/hacking

After achieving this, Wagner then turned his attention to the core application on the device - a binary called `ppsapp`.  He managed to successfully reverse engineer this binary and document the steps required to patch `ppsapp` to enable rtsp streaming.

Wagner's [patching guide](https://github.com/guino/ppsapp-rtsp/) is very comprehensive, and I tried valiantly to follow his steps on my own copy of `ppsapp` but I was most certainly out of my depth.

Fortunately, Wagner is kindly patching and [hosting](https://github.com/guino/ppsapp-rtsp/issues/1) various versions of the file and I **STRONGLY** recommend you check there first, before trying to patch the file on your own.  Be sure to confirm exact version using the included md5 hashes.  I accepted Wagner's offer to patch my `ppsapp`, which can be found [here](https://github.com/guino/ppsapp-rtsp/issues/1#issuecomment-769365380).

### Why didn't the entire process work for me?

When trying to use Wagner's process, I found that my device didn't like running the patched version of the file.  In fact, it didn't even like it when I killed the original process and re-ran the unpatched version from the SD card.

After conferring with Wagner, we concluded there must be something about an in-built watchdog process that doesn't like it when the app is executed after init, so the key was to try and get the camera to execute our patched `ppsapp` during init.

### How did I get it working?

After a *lot* of trial and error, I finally managed to get my camera to successfully run the patched `ppsapp` during init, by varying Wagner's process per the below:

1. I put a completely rewritten version of the `S90PPStrong` init script in the root of the SD card (named `S90PPStrong-290`).

2. I varied [step 2](https://github.com/guino/BazzDoorbell/issues/2) by including an additional command in my `env` file to copy the `S90PPSAtrong-290` init script to `/etc/init.d/S90PPStrong`. This had to be done *very early* in the init process - trying to use `initrun.sh` to do this was simply too late - the stock init.d script was being executed before we could overwrite it.

3. I placed the patched `ppsapp` in the root of my SD card, named `ppsapp-rtsp` (so as not to clash with `custom.sh` in Wagner's hack).

You can find the last two items [here](./mmc).  In short, I added the following the `env` file, just before the command the executes `/mnt/mmc/initrun.sh`:

```
cp_/mnt/mmc01/S90PPStrong-290_/etc/init.d/S90PPStrong;
```

So, my complete `env` file looks like:

```
bootargs=mem=37M console=ttyAMA0,115200n8 mtdparts=hi_sfc:384k(bld)ro,64k(env),64k(enc)ro,64k(sysflg),3584k(sys),6656k(app),1536k(cfg),1m(recove),2880k(user),128k(oeminfo) ppsAppParts=5 ppsWatchInitEnd - ip=\\${T//_/\\$\\'"\\\\x20"\\'}:::::";T=\\"sleep_5;mkdir_-p_/mnt/mmc01;mount_-t_vfat_/dev/mmcblk0p1_/mnt/mmc01;cp_/mnt/mmc01/S90PPStrong-290_/etc/init.d/S90PPStrong;/mnt/mmc01/initrun.sh&\\";eval"
```

**NOTE:** there are important characters that would be missing from the end of the file if you were to just copy/paste this text.  Wagner explains this after step 2 in [his guide](https://github.com/guino/BazzDoorbell/issues/2).  Also, don't just use my `env` file, unless your bootargs are already identical - also described in Wagner's guide.

### Conclusion

Major thanks to Wagner for all his hard work and perseverance, as well his generosity in patching people's `ppsapp` files for them.  Were it not for this, I'd still be mucking around with a Raspberry Pi hanging off the UART pins, trying in vain to discover the bootloader password.

I know I said it earlier, but I'll say it again.  If you benefit from this, please consider [shouting Wagner a coffee](http://paypal.me/wbbo) for all his hard work.

Cheers!