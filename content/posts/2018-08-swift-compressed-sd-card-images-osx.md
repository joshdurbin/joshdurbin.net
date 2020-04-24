+++
title = "Reading and writing compressed disk images with dd in MacOS"
description = "Reading and writing compressed disk images with dd and rdisk in MacOS"
date = "2018-08-23"
tags = ["macos"]
+++

I've been prepping server images for my offline journey running the lights on the fabulous gay sheep at Burning Man 2018.
Doing so requires yearly updates, enhancements, new server builds and copies of old ones just in case the shit hits the
fan without access to the outside world. This year I decided to write scripts to make things easier, efficiently store image data,
and quickly read/write it within MacOS.

The following scripts are what came out of the efforts:

The first is meant to capture data from an inserted SD card by listing the mounting disks and volumes. One enhancement,
change from the RPI docs, is to use the rdisk reference to the target disk instead of the plain disk. The reason for this
is explained [here](https://superuser.com/questions/631592/why-is-dev-rdisk-about-20-times-faster-than-dev-disk-in-mac-os-x).

The backup script reads the disk in 1 megabyte (ish) blocks, piping the data to gzip where empty nodes are dropped from the image,
saving a lot of space, particularly with large SD cards.

```
#!/usr/bin/env bash

diskutil list

read -p "Enter device name (for example for device /dev/disk2 enter disk2): " device
diskutil unmountDisk /dev/$device

write_location=~/baaahs_lights_server_`date +%d%m%Y`.img
echo
echo "Writing compressed image to $write_location"
sudo dd bs=1m if=/dev/r$device | gzip > $write_location
```

The second script does the opposite, writes in 4 megabyte (ish) blocks and unlike the previous script, must be run as root.

```
#!/usr/bin/env bash

diskutil list

read -p "Enter device name (for example for device /dev/disk2 enter disk2): " device
diskutil unmountDisk /dev/$device

read -p "Enter the gzip compressed image path: " image_path
gzip -dc $image_path | dd bs=4m of=/dev/r$device
```

...allowing for quick capture and restore of server images. Woot.