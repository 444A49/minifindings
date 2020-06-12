# Minifindings

This is a reference file of all the tips, tricks, and hacks for the Mavic Mini I've found somewhere else, discovered on my own, ~~or stolen from someone else~~. I'll document everything I'll find, but ultimately I have three main goals:

1. Force the RC to remain in the FCC hack, it's very annoying to do the FCC hack dance everytime I want to fly the mini.
2. Silence the RC, without screwing a screw into the speaker. It's loud, annoying, and draws unwanted attention
3. Disable NFZ, DJI should not be the one deciding if I'm allowed or not to fly the drone.

All my research is done on my own drone, which is in firmware .200. I don't know which version the RC is running, but I'm guessing it's the first one, or the one that was released along AC .200

## Tools

Most of the hacks and tweaks will be done with Original Gangsters' [dji-firmware-tools](https://github.com/o-gs/dji-firmware-tools/). Make sure you have python3 installed in your system. I'm using Ubuntu and all the commands will assume you are in a \*nix system.

I strongly recommend you joining the [dji-rev slack group](https://dji-rev.slack.com/). You can find an invitation link [in their wiki](https://dji.retroroms.info/). They are usually highly secretive of their findings but sometimes they drop firmware dumps and useful information.

## Glossary

 * AC: Aircraft
 * RC: Remote controller
 * DUML: DJI Unified Markup Language. It's basically the language that the RC/AC talk.

## AC

### Firmware dumps

I was lucky enough to catch a firmware dump of version .400 in the dji-rev slack channel. I cannot share it here, but if you ask around, you might get it.

The mini runs on a modified version of Android.

### Serial mode

The mini has a serial mode that has to be manually enabled on each boot. If you plug the mini to your computer and wait a few seconds, it will present itself to the pc as a mass storage drive. Serial mode can be enabled _before_ this mass storage mode is enabled. This can be easily done by sending the GetVersion command with `comm_og_service_tool.py`.

```./comm_serialtalk.py /dev/ttyACM0 --cmd_set=0 --cmd_id=1 --sender=1001 --receiver_type=31```

 GetVersion is command 1 of set 0. Why the `sender` and `receiver_type` options are needed, I have no idea. I asked in the Slack channel and I was told that those parameters are required. So there's that.
 
It might be wise to wrap this into a script that checks the availability of `/dev/ttyACM0` as you might run the command too early in the booting process, which will give you the `No such file or directory: '/dev/ttyACM0'` error. Or you can just run the command endelessly until you get a hexadecimal response.

If you get permission errors, run the command as `sudo` or add your user to the `dialout` group.

Once you get a response, the drone will remain in serial mode, will expose the FTP and accept parameters. It will refuse to teak off and the app will suggest you to restart the drone. Unplugging it does the trick as well.


### FTP

FTP is enabled if the AC is in serial mode. Turn on the AC, plug it to your computer, put it in serial mode and FTP into 192.168.1.3, port 23. Username is root, blank password.

These credentials can be found in the `/etc/ftp.conf` file.

All the files that are exposed in the FTP are encrypted and you cannot exit the `/tmp/blackbox` folder. FTP service seems to be driven by the `dji_ftpd` file. If you dissasemble it, there's a function that performs a couple of checks before `cd`ing to the folder you command it to. It will check for `/..`, `//`, and length of the requested path before even attempt changing the current directory to it. It will also prepend all the paths given by the user with `/tmp/blackbox` before calling `realpath`.

Inspecting the way the `/tmp/blackbox` folder is initialized (there are a couple of shell files in `/etc/init.d` that run on startup - BTW, you'd be suprised by the amount of shell files that command different parts of the drone) it seems that there's no encryption on the files at write time, but rather at read time, and seems that its managed by either `dji_blackbox` or `dji_ftpd`.

### Parameters

Besides what the fly app allows you to configure, the Mini exposes a couple of parameters that can only be tweaked on serial mode. You can get a list of these parameters with the `comm_og_service_tool.py` script. To get the full list, simply run `./comm_og_service_tool.py  /dev/ttyACM0 WM160 FlycParam list --count 1500`. The Mini exposes one table with around 650 parameters (parameters are exposed in tables with sets of parameters, the Mini happens to have only one), although you'll see that `comm_og_service_tool.py` throws a couple of errors while reading the parameters. It seems that the Mini reports around 1500 parameters, but only lets you access a third of that.

#### Downward speed on sport mode

`vert_vel_down_adding_max_0 -7` # Maximum general downward speed

`g_config.mode_sport_cfg.vert_vel_down_0 -7` # Maximum downward speed

`config.mode_sport_cfg.vert_vel_up_0 4` # Maximum upward speed

You have to be very careful with these values because you can easily crash it when landing it, if you don't let the mini slow down first. The normal sport mode makes it descend really slow; switching the values to -7 won't prevent it from smashing the ground. If you think you can handle it, try `-10`. It can reach 40 km/h while going down. It's scary.

#### Minimum hovering distance

`min_height_user` # Sets the minimum hovering distance to a few centimeters. Hovering might be unstable on non-well lit areas. Must be issued with `--alt`. Credit to Bob Maker @ dji-rev.

To set a param, issue the following command:

`./comm_og_service_tool.py /dev/ttyACM0 WM160 FlycParam set #parameter_name #new_value`

Parameters have a min and max value and going outside those values has no effect.

### FCC hack

The Mini, like most DJI drones, have two configurations that are activated based on location: FCC and CE. You want to use always the FCC mode, which is only available in USA. This allows you to use the full power throughput of the RC and AC.

There are plenty of tutorials about this on Youtube. It basically boils down to trick the Fly App into thinking it's in an FCC area, connecting to the drone while shielding it so it cannot acquire GPS lock, and then taking off. 

## RC

The RC accepts DUMLs if you connect it to a computer via a normal micro USB to USB cable. There's no serial mode to initialize and there's no timeout. It happily accepts packets even after minutes of being on.

### Speaker

The RC has an extremely annoying speaker that will beep at you for a varaiety of reasons. There's no way to silence it via software, but on mavicpilots I've seen tutorials showing that a screw directly to the speaker dampens its volume.

### Shell access

You can easily shell the RC by plugging a USB to Ethernet adapter. Plug the ethernet cable to your PC, set `192.168.3.2` as your PC IP address, and `telnet` to 192.168.3.10. It gives you root access.

### Firmware

RC's firmware is completely unencrypted. You can download it from the [Dank Drone Downloader](http://dankdronedownloader.co.uk/DDD2/app/) website and extract the files with [binwalk](https://github.com/ReFirmLabs/binwalk). 

### Dumping

You can easily dump all the files by creating a `tar` file and sending it over the network with `nc`. First, on your PC, put `nc` in listening mode, dumping what it receives to a file:

`nc -l -p 1234 > rc.tar`

This will open port `1234` in your PC and create a file named `rc.tar` that you can uncompress later. Then, in the RC, run this command:

`tar -cv --exclude="sys" --exclude="proc" / | nc 192.168.3.2 1234`

This will `tar` all the contents of the root filesystem, excluding `sys` and `proc` and pipe it over `nc`. Then you can uncompress the file in your PC.

A better and more complete approach would be `cat`ing `/dev/mtd*`, which in theory will give you the full filesystem, but in practice the dumps that this create are incomplete; for example, they lack of the `tmp` folder.

#### Files

* `/usr/bin/apsrv`:  (APplication SeRVice?) Seems to be in charge of the communication between phone and RC. If you run `strings` on it you'll see plenty of `CMDID_*` strings that could be potentially triggered with DUML packets. It also has interesting functions like `switch_mode_based_on_country` and `set_disable_country_flag`
* `/etc/amt/wifi.conf`: This one is the one that changes when doing the FCC hack. In FCC mode, `country` holds the value `US`. It also holds other information about the RC connection itself:
  * `rc_ssid`: For some reason, in my RC it's `Spark-RC-XXXXX`. Leftovers from Spark's firmware?
  * `rc_passwd`: It's `12341234`
  * `disable_country`: boolean, always set to 0, in CE or FCC mode. Related to the FCC hack?
  * `sky_passwd`: Unlike `rc_passwd` this one is random and changes when you resync the RC with the AC (holding the power button for 4 seconds on the AC while on)

### Silencing the RC

Seems that the speaker is controlled by something running outside the linux environment exposed by the ethernet adapter. This can be easily verified by issuing `pkill '^'` on the RC's shell, which will kill all processes and triggering a restart of the OS. The speaker still beeps during the restart (if you press buttons), hinting that this is managed by another layer of the RC.

## Fly app

### Offline mode with maps

I don't like the idea letting DJI know everything I do with the drone, so I have a dedicated phone for it, which is on airplane mode all the time. The only downside is that you have to 1) login once to a fake account, 2) precache the maps.

Install the app, create a fake account (you don't even need a real one, they don't send confirmation emails) and enter camera view. Then click on the map on the lower left corner and browse around all the areas you want to cache. Make sure you zoom in and out. Then switch to airplane mode.

If you need to keep the phone connected to the network, you can do two things:

1) Install [afwall](https://github.com/ukanth/afwall) and deny access to the app.
2) Push the following domains to the phone's `/etc/hosts` file (you'll need root):

```
0.0.0.0         flysafe-api.dji.com
0.0.0.0         dji.com
0.0.0.0         www.dji.com
0.0.0.0         developer.dji.com
0.0.0.0         mydjiflight.dji.com
0.0.0.0         account-api.dji.com
0.0.0.0         api.djiservice.org
0.0.0.0         sentry.djiops.com
0.0.0.0         app-h5.dji.com
0.0.0.0         active.dji.com
0.0.0.0         statistical-report.djiservice.org
0.0.0.0         click.dji.com
0.0.0.0         mimo.skypixel.com
0.0.0.0         events.mapbox.com
```

This list [was found on reddit](https://www.reddit.com/r/djimavicmini/comments/e2nerb/dji_domains_blocklist/).

### NFZ database

The app's NFZ database can be easily removed by deleting a couple of files. You'll need a rooted phone. This won't allow you flying into a NFZ (the drone keeps a copy of the database) but at least you won't see the annoying overlays of authorization zones, no fly zones, etc. on the map. Simply delete the following folder:

* `/data/data/dji.go.v5/files/flysafe`
