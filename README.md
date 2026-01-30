# My notes on my FLSun S1 Pro

## Hardware

Overall, the hardware is pretty darn good. 36VDC.

Delta calibration and `DELTA_ANALYZE` show top-notch build quality with good precision:

```
[printer]
delta_radius = 182.802597

[stepper_a]
angle = 209.631916
arm_length = 383.394068
position_endstop = 431.100973

[stepper_b]
angle = 329.662331
arm_length = 385.519306
position_endstop = 430.488460

[stepper_c]
angle = 90.000000
arm_length = 383.153061
position_endstop = 430.122331
```

The CHT-style geometry in the hotend seems to make it very hard to impossible to eliminate stringing using standard methods.
I'm likely going to see if I can kill it with gofast, but no luck so far.  I'm planning to put a
[Chube Air](https://chubehotends.com/) and an [LGX Lite](https://www.bondtech.se/product/lgx-lite-large-gears-extruder/)
in there relatively soon.

There's a hardened steel nozzle, but the heat block with the CHT geometry in it seems to be aluminum, so that would blow out
and need replacement very quickly if you actually printed anything that needed hardened steel.  I've swapped mine for an
[Undertaker nozzle](https://west3d.com/products/west3ds-undertaker-tungsten-carbide-nozzle) from West3D.

### Build plate

Diameter is 330mm  
Tab is 90mm wide with broad radius  
Diameter + tab is 350mm  

### Chamber Temps

So far I've only printed PLA, but it's pretty clear the stock chamber will never get toasty enough for really good ABS
printing, and not even close for decent PC printing. I expect to insulate the walls in the inside, likely using PIR panels,
and be sure to get as much of the motor as possible in the "cold" part of the electronics bay at the top.

I also plan to have the turbo fan pull air from the chamber rather than outside.  I hate PLA, so I don't need the insane
level of cooling the existing setup can get, I value the chamber temp more. This will likely kill the turbo fan and I'll
have to find some way of moving hot air.

### Issues

I had an issue where the A motor would skip steps and the printer would shut down. Recalibrating the motors fixed it,
hopefully for good.

## Software

### Linux

Running an old buster, for now you can change the URLs in `/etc/apt/sources.list` to start with `http://archive.debian.org/`
and apt and friends will still work for a while.

`/etc/profile.d/bash_completion.sh` ends with the line `bash`.  This causes bash to start another bash instance while
running the main profile (/etc/profile) and *before* the users profile (~/.profile) is parsed.  If you log in, you need
to `exit` the shell you're in before your profile completes.  If you `exit` a second time, you actually log off.
Just removing that line fixes the problem. However, `apt reinstall bash-completion` doesn't for some unknown reason.

`lsusb` is a garbage file. `apt reinstall usbtools` fixes it.

`/etc/rc.local` does some crazy crap, removing the following lines is a good idea:
```
chown -R pi:pi /home/pi
chmod 777 -R /home/pi
chmod 777 -R /home/pi/printer_data/config_bak
chmod 777 -R /home/pi/printer_data/config
```

The first one isn't completely gross, but the other three are. Notably, the second one makes it so you can't log in over
SSH using a public key.

Once you update to the latest firmware, `sshd` is running with a user of `pi` and a password of `1`.  Further, logging in
as pi, I couldn't use `passwd` to change my password but luckily `pi` is in the `sudo` group which is in `/etc/sudoers`,
and `sudo passwd pi` worked.  I didn't dig out what the root password was, but I changed that as well just in case.

### Klipper

[`DELTA_ANALYZE`](https://www.klipper3d.org/Delta_Calibrate.html) broken due to running python3 and not having
[commit 7f9ea23](https://github.com/Klipper3d/klipper/commit/7f9ea231b7b9e0f76bf2c5ec02f8af89e7017f76)
present.  Manually making the change fixes `DELTA_ANALYZE`.

There's some magic PA calculator thing.

#### Config

There's a laser pin in the config.  This looks like something from the pre-Pro S1 that's not there anymore.

The `run_current` for the extruder stepper is 1.2A, which seems unlikely to be a good idea.  Changed to 0.8 as in the
Guilouz printer.cfg.  This *still* seems on the high side for what the motor is, but I haven't dug in and found the motor
plate yet.

Default config has an extruder `hold_current` which is generally not a good idea because detent forces can cause uncommanded
movement when changing current from one to the other. TODO: Just turn off the extruder motor when not printing/extruding.

Various things are saved in `~/savedVariables1.cfg`. I went looking for this to find where the Z offset was stored.

### Mainsail

The version of the push streamer crashes periodically, and it's clear someone knew about this since there's a restart thing
running as video9-device.service.
`ExecStart=/bin/sh -c "while true; do if [ -e /dev/video9 ]; then systemctl start pushStream.service; else systemctl stop pushStream.service; fi; sleep 5; done"` 
Presumably updating would solve the issue.

The hardware has support for video encoding, but Mailsail is using MJPEG for the webcam.

There's some FLSun changes in `~/klipper/klippy/extras/bed_mesh.py` that adds extra probe points around the edge of the bed,
but Mainsail does not seem to have a corresponding change, resulting in the Mainsail heightmap being basically useless.

### References

[FLSun S1 Pro "wiki"](https://wiki.flsun3d.com/en/S1Pro)  
[FLSUN S1 Open Source Edition](https://guilouz.github.io/FLSUN-S1-Open-Source-Edition/about/)  
[Guilouz printer.cfg](https://github.com/Guilouz/Klipper-Flsun-S1/blob/master/config/FLSUN%20S1/printer.cfg)
