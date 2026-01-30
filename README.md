# My notes on my FLSun S1 Pro

## Hardware

## Software

### Linux
Running an old buster, for now you can change the URLs in `/etc/apt/sources.list` to start with `http://archive.debian.org/`
and apt and friends will still work for a while.

`/etc/profile.d/bash_completion.sh` ends with the line `bash`.  This causes bash to start another bash instance while
running the main profile (/etc/profile) and *before* the users profile (~/.profile) is parsed.  If you log in, you need
to `exit` the shell you're in before your profile completes.  If you `exit` a second time, you actually log off.
Just removing that line fixes the problem. However, `apt reinstall bash-completion` doesn't for some unknown reason.

`lsusb` is a garbage file. `apt reinstall usbtools` fixes it.

`/etc/rc.local` does some crazy crap, removingthe following lines is a good idea:
```
chown -R pi:pi /home/pi
chmod 777 -R /home/pi
chmod 777 -R /home/pi/printer_data/config_bak
chmod 777 -R /home/pi/printer_data/config
```

The first one isn't completely gross, but the other three are. Notably, the second one makes it so you can't log in over
SSH using a public key.


### Klipper
DELTA_ANALYZE broken due to running python3 and not having commit 7f9ea23 present.  Manually making the change fixes
DELTA_ANALYZE.

### Moonraker

### Mainsail
