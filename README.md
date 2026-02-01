# My notes on my FLSun S1 Pro

I've started using Kalico with the Open Source Edition.

## Open Source Edition

### Enhanced calibrate

Using `DELTA_CALIBRATE ENHANCED_METHOD=1` gets me results that appear to be slightly better than the `DELTA_ANALYZE` on my
printer.

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

The CHT-style geometry in the hotend seems to make it a bit more difficult to eliminate stringing. Keep at it, a bit of
retract and wipe will clear it up with enough work.

I'm planning to put a
[Chube Air](https://chubehotends.com/) and an [LGX Lite](https://www.bondtech.se/product/lgx-lite-large-gears-extruder/)
in there relatively soon.

There's a hardened steel nozzle, but the heat block with the CHT geometry in it seems to be aluminum, so that would blow out
and need replacement very quickly if you actually printed anything that needed hardened steel.  I've swapped mine for an
[Undertaker nozzle](https://west3d.com/products/west3ds-undertaker-tungsten-carbide-nozzle) from West3D.

### Build plate

Diameter is 330mm  
Tab is 90mm wide with broad radius  
Diameter + tab is 350mm  

#### Load cells

Bed mesh probing is *very* variable.  I've done a highly precise bed mesh using tight tolerance and many many retries,
If this works, it should be rolled into the BED_LEVEL_2 macro.

##### Klipper shutdown

When I run `PROBE_ACCURACY`, I get the following error and Klippy restarts:
```
[2026-01-30 16:46:02,638][WARNING] Internal error on command:"PROBE_ACCURACY"
[2026-01-30 16:46:02,648][ERROR] Internal Error on WebRequest: gcode/script
Traceback (most recent call last):
  File "/home/pi/klipper/klippy/webhooks.py", line 252, in _process_request
    func(web_request)
  File "/home/pi/klipper/klippy/webhooks.py", line 428, in _handle_script
    self.gcode.run_script(web_request.get_str('script'))
  File "/home/pi/klipper/klippy/gcode.py", line 223, in run_script
    self._process_commands(script.split('\n'), need_ack=False)
  File "/home/pi/klipper/klippy/gcode.py", line 205, in _process_commands
    handler(gcmd)
  File "/home/pi/klipper/klippy/gcode.py", line 138, in <lambda>
    func = lambda params: origfunc(self._get_extended_params(params))
  File "/home/pi/klipper/klippy/extras/probe.py", line 236, in cmd_PROBE_ACCURACY
    pos = self._probe(speed)
TypeError: _probe() missing 2 required positional arguments: 'gcmd' and 'samples_retries'
```

Fix:

```
--- klippy/extras/probe.py.flsun	2026-01-30 17:35:15.898430548 -0500
+++ klippy/extras/probe.py	2026-01-30 17:36:03.555003856 -0500
@@ -233,7 +233,7 @@
         positions = []
         while len(positions) < sample_count:
             # Probe position
-            pos = self._probe(speed)
+            pos = self._probe(speed, gcmd, 0)
             positions.append(pos)
             # Retract
             liftpos = [None, None, pos[2] + sample_retract_dist]
```

It appears that FLSun has done some stuff to probing that needs to be analyzed.

### Chamber Temps

So far I've only printed PLA, but it's pretty clear the stock chamber will never get toasty enough for really good ABS
printing, and not even close for decent PC printing. I expect to insulate the walls in the inside, likely using PIR panels,
and be sure to get as much of the motor as possible in the "cold" part of the electronics bay at the top.

I also plan to have the turbo fan pull air from the chamber rather than outside.  I hate PLA, so I don't need the insane
level of cooling the existing setup can get, I value the chamber temp more. This will likely kill the turbo fan and I'll
have to find some way of moving hot air.

### Drybox

It appears that the fan and heater are controlled via pins on the MCU, but status is read via /dev/ttyS4 in a json-rpc
format at 19200, 8-N-1.

```
{"jsonrpc":"2.0","result":{"drybox":{"temperature":27.0,"humidity":21.0,"weight":20}},"id":1}
```

### Issues

I had an issue where the A motor would skip steps and the printer would shut down. Recalibrating the motors fixed it,
hopefully for good.

About one in three times when I load a spool of filament, the end catches in the sensor and flexes the PTFE tube out of the
drybox. It looks like a simple clip to hold the tube straight will keep that from happening.  Cleap available on [Printables](https://www.printables.com/model/1579001-dry-box-feed-tube-support-for-flsun-s1-pro)

While I'm printing large so the outer heater is used, the outer heater sits around 100% and the inner one is usually zero
and occasionally bumps up to 8% for half a second. I need to take a closer look at this.

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

Seems to be from around `06a31222` (Jun, 2022)

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

### Screen interface

The Bed Level calibration does a `DELTA_CALIBRATE` so if you just want to update your mesh, it's the wrong thing. Use the
`BED_LEVEL_2` macro from Mainsail.

## References

[FLSun S1 Pro "wiki"](https://wiki.flsun3d.com/en/S1Pro)  
[FLSUN S1 Open Source Edition](https://guilouz.github.io/FLSUN-S1-Open-Source-Edition/about/)  
[Guilouz printer.cfg](https://github.com/Guilouz/Klipper-Flsun-S1/blob/master/config/FLSUN%20S1/printer.cfg)
