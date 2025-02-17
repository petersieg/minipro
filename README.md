# minipro

minipro is an open source program for controlling the MiniPRO TL866xx series of chip programmers. The source code is available at https://gitlab.com/DavidGriffith/minipro/

This repository is my notes on how to use it with a [TL866II Plus](http://www.xgecu.com/en/TL866_main.html) universal programmer on openSUSE.

![Image of TL866II Plus](resources/tl866iiplus.png)


## Building

Build instructions for openSUSE have been added to the official repository and are no longer available here. Otherwise:

sudo apt-get install build-essential pkg-config git libusb-1.0-0-dev fakeroot debhelper dpkg-dev

git clone https://gitlab.com/DavidGriffith/minipro.git

cd minipro

fakeroot dpkg-buildpackage -b -us -uc

sudo dpkg -i ../minipro_0.7.1_amd64.deb  --  (or whatever the actual version might be by at the time - find resulting deb package here in repo)

Instead of git and git clone, one can also just download zip archive, extract it and cd into directory.


## How to use

Beware of using the programmer connected to an unpowered USB-hub. It draws a lot of power in use, and can potentially damage chips.

Start by finding the correct name of the device for use with the software. You can search using the `-L` parameter like this: 

`$ minipro -L atmega328`

The concrete device could be a `ATMEGA328P@DIP28` in this case. Include this with the `-p` parameter for all future operations.

Insert the chip with pin 1 (see the dot) pointing towards the lever of the ZIF socket:

![Chip orientation in the TL866II Plus](resources/chip-orientation.png)

(_Image taken from the Windows software_)


### Working with (E)EPROMs

Read the contents of the EEPROM and save it to the file eeprom.bin:

`$ minipro -p "AT28C16@DIP24" -r eeprom.bin`

Write the contents of the file eeprom.bin to the EEPROM:

`$ minipro -p "AT28C16@DIP24" -w eeprom.bin`

Verify that the EEPROM has the same contents as the file eeprom.bin:

`$ minipro -p "AT28C16@DIP24" -m eeprom.bin`

Blank check (-b) with pin check (-z) for good contact:

`$ minipro -p "M27C64@DIP28" -b -z`


### Working with AVR

AVR microcontrollers may have both flash memory for the program and EEPROM for the data, in addition to configuration parameters in the form of "fuses". All these can be read and written to, unless certain lock bits have been set. Lock bits can stop you from reading the chip, but erasing the chip will also reset the lock bits.

Writing to the chip will by default erase the chip. This can be confusing when you want to write to the different parts of the chip, as writing one of them will blank the other part. This can be avoided by erasing the chip first with the `-E` parameter, and then specifying the `-e` parameter when writing to disable automatic erase.

Erase chip:

`$ minipro -p "ATMEGA328P@DIP28" -E`

Write the contents of the file eeprom.bin to the EEPROM:

 `$ minipro -p "ATMEGA328P@DIP28" -c data -w eeprom.bin -e`

Write the contents of the file program.bin to the flash memory: 

`$ minipro -p "ATMEGA328P@DIP28" -c code -w program.bin -e`

Write the configuration for fuses and lock bits specified in the file fuses.cfg to the chip:

`$ minipro -p "ATMEGA328P@DIP28" -c config -w fuses.cfg -e`

This is the format for fuses.cfg:

```
fuses_lo = 0x62
fuses_hi = 0xd9
fuses_ext = 0xff
lock_byte = 0xff
```

Read the datasheet or use this http://www.engbedded.com/fusecalc/ to understand what this means.

Like the examples from working with EEPROM, you can verify the contents with the `-m` parameter, and read the contents into a file with the `-r` parameter:

```
$ minipro -p "ATMEGA328P@DIP28" -r firmware.bin
Found TL866II+ 04.2.111 (0x26f)
Chip ID OK: 0x1E950F
Reading Code...  0.66Sec  OK
Reading Data...  0.04Sec  OK
Reading fuses... 0.00Sec  OK

$ ls -la
-rw-r--r--  1 blurpy users  32768 jan.  22 20:12 firmware.bin
-rw-r--r--  1 blurpy users   1024 jan.  22 20:12 firmware.eeprom.bin
-rw-r--r--  1 blurpy users     66 jan.  22 20:12 firmware.fuses.conf
```

Beware that the chip might appear completely empty if the fuses are configured for read protection. It will look like this:

```
$ hexdump firmware.bin 
0000000 ffff ffff ffff ffff ffff ffff ffff ffff
*
0008000
```


### Working with AVR using In Circuit Serial Programming (ICSP)

The TL866II Plus has an ICSP port on the side of the unit that allows you to flash a chip without removing it from the circuit board. A cable for connecting to the port should also be in the box.

This is the pinout:

![ICSP connector on the TL866II Plus](resources/icsp-connection.png)

(_Image taken from the Windows software_)

Many Arduino devices come with a 6-pin ICSP header like this:

![ICSP header](resources/icsp-header.png)

Here is an example of connecting the TL866II Plus to an Arduino Nano clone (ks0173) using ICSP:

![ICSP example connection](resources/ks0173-icsp.jpg)

There are a couple of things to note:

1. The pinout on different boards might not have the same orientation or order. Check with a multimeter to make sure you connect the wires correctly.
2. The TL866II Plus outputs 5 volts on VCC. Some boards might only tolerate 3.3 volts and could fry. Power the board through other means to avoid that, or maybe try lowering the voltage using resistors (not tested).

The commands are very similar to when you have a chip in the ZIF socket. You only need to add `-i`, like this:

```
$ minipro -p "ATMEGA328P@TQFP32" -r ks0173.bin -i
Found TL866II+ 04.2.111 (0x26f)
Activating ICSP...
Chip ID OK: 0x1E950F
Reading Code...  7.19Sec  OK
Reading Data...  0.23Sec  OK
Reading fuses... 0.00Sec  OK
```

Result:

```
$ ls -la
-rw-r--r-- 1 blurpy users  32768 jan.  23 13:56 ks0173.bin
-rw-r--r-- 1 blurpy users   1024 jan.  23 13:56 ks0173.eeprom.bin
-rw-r--r-- 1 blurpy users     66 jan.  23 13:56 ks0173.fuses.conf
```

You can use `-I` instead of `-i` to use ICSP with VCC disabled.


### Programming GALs

The ATF22V10C GALs are supported by the same cheap XGecu TL866IIplus programmer. Note that you need the new version TL866II+. The old TL866 cannot program those chips. 
You do need to compile the described logic equations (in a .pld file), with Atmel's WinCUPL program, to get output programming file (.jed).
Or try the open source galasm: https://github.com/hanshuebner/galasm 
The .jed file must be programmed into the GALs. This can be done using minipro:

    $ minipro -p atf22v10c -w file.jed  


### Testing logic ICs

```
ich@ich-iMac27:~$ minipro -L 74153
Found TL866II+ 04.2.132 (0x284)
Device code: 19360167
Serial code: TOEVAR0TKXQ9Y4I7Q206
USB speed: 12Mbps (USB 1.1)
74153
ich@ich-iMac27:~$ minipro -p 74153 -T
Found TL866II+ 04.2.132 (0x284)
Device code: 19360167
Serial code: TOEVAR0TKXQ9Y4I7Q206
USB speed: 12Mbps (USB 1.1)
      1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16
0000: 1  0  0  1  0  1  L  G  L  1  0  1  0  1  1  V
0001: 1  1  1  0  1  0  L  G  L  0  1  0  1  0  1  V
0002: 0  0  1  0  1  0  L  G  L  1  0  1  0  0  1  V
0003: 0  0  0  1  0  1  H  G  L  0  1  0  1  0  1  V
0004: 0  0  0  1  0  1  L  G  L  1  0  1  0  1  1  V
0005: 0  1  1  0  1  0  H  G  L  0  1  0  1  1  1  V
0006: 0  1  1  0  1  0  L  G  L  1  0  1  0  0  1  V
0007: 0  1  0  1  0  1  H  G  L  0  1  0  1  0  1  V
0008: 0  1  0  1  0  1  L  G  L  1  0  1  0  1  1  V
0009: 0  1  1  0  1  0  H  G  L  0  1  0  1  1  1  V
0010: 1  0  0  1  0  1  L  G  L  0  1  0  1  0  0  V
0011: 1  0  1  0  1  0  L  G  H  1  0  1  0  0  0  V
0012: 1  0  0  1  0  1  L  G  L  1  0  1  0  1  0  V
0013: 1  0  1  0  1  0  L  G  H  0  1  0  1  1  0  V
0014: 1  1  0  1  0  1  L  G  L  1  0  0  1  0  0  V
0015: 1  1  1  0  1  0  L  G  H  0  1  1  0  0  0  V
0016: 1  1  0  1  0  1  L  G  L  1  0  1  0  1  0  V
0017: 1  1  1  0  1  0  L  G  H  0  1  0  1  1  0  V
Logic test successful.
ich@ich-iMac27:~$
```

### Hardware verifications

You can test for good pin connections with a particular chip inserted in the programmer:

`$ minipro -p AT28C256 -z`

To test that all the pins in the programmer works as intended, you can run a full hardware self check. Be sure to remove any chips from the programmer, otherwise you risk damaging it.

`$ minipro -t`


### Update firmware

You wont find the firmware as a standalone download, so you need to download the Windows software `Xgpro` from the homepage. If the homepage is too slow, there is a mirror here: https://github.com/Kreeblah/XGecu_Software

Unrar the downloaded file to find a Windows executable. Unrar the executable to find the firmware file `updateII.dat`. Both `unar` and `unrar` are examples of tools that can extract the files.
 
 ```
$ unar XgproV1181_Setup.rar
$ unar XgproV1181_Setup.exe
```

 ```
$ unrar x XgproV1181_Setup.rar
$ unrar x XgproV1181_Setup.exe
```

Update the firmware like this:

`$ minipro -F updateII.dat`


#### Troubleshooting

I tried to update the firmware using a virtual machine with Windows and the official Xgpro software once. The programmer was functioning fine in the VM, but updating the firmware failed with no reason given. The programmer did not function anymore after that. Any command issued with minipro gave me the message "_Found TL866II+ in bootloader mode_" and quit. After reading this [issue](https://gitlab.com/DavidGriffith/minipro/issues/148) it looked like the firmware update process in minipro is much more reliable than the official software, so I tested, and the update completed successfully, and the programmer was once again working. 
