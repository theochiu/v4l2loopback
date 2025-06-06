v4l2loopback - a kernel module to create V4L2 loopback devices
==============================================================

This module allows you to create "virtual video devices".
Normal (v4l2) applications will read these devices as if they were ordinary
video devices, but the video will not be read from e.g. a capture card but
instead it is generated by another application.
This allows you for instance to apply some nifty video effects on your
Skype video...
It also allows some more serious things (e.g. I've been using it to add
streaming capabilities to an application by the means of hooking GStreamer into
the loopback devices).

# NEWS
To get the main features of each new release, see the NEWS file.
You could also have a look at the ChangeLog (which gets automatically generated and might
only be of limited use...

# ISSUES
For current issues, checkout https://github.com/umlaeute/v4l2loopback/issues
Please use the issue-tracker for reporting any problems.

Before you create a new ticket in our issue tracker, please make sure that you have read
*this* document and followed any instructions found within.

Also, please search the issue-tracker *before* reporting any problems: it's much better
to add your information to an existing ticket than to create a new ticket with essentially
the same information.

## SEEKING HELP
The issue tracker is meant to track specific bugs in the code (and new features).
However, it is ill-suited as a user support forum.

If you have general questions or problems, please use the `v4l2loopback` tag
on [Unix & Linux](https://unix.stackexchange.com/questions/tagged/v4l2loopback) instead:
https://unix.stackexchange.com/questions/tagged/v4l2loopback


# DEPENDENCIES
In order to build (compile,...) anything, you must have a *working* build-environment
(compiler, GNU make,...).
The kernel can be somewhat picky if you try to load a module that was compiled with
a different compiler as was used to compile the kernel itself.
So make sure to have the right compiler in place.

The v4l2loopback module is a *kernel module*.
In order to build it, you *must have* the kernel headers installed that match
the linux kernel with which you want to use the module (in most cases this will
be the kernel that you are currently running).
Please note, that kernel headers and kernel image must have *exactly the same* version.
For example, `3.18.0-trunk-rpi` is a different version than `3.18.7-v7+`, even though
the first few numbers are the same.
(Modules will be incompatible if the versions don't match. If you are lucky, the module will
simply refuse to load. If you are unlucky, your computer will spit in your eye or do worse.)

There are distribution-specific differences on how to get the correct kernel headers
(or to install a compilation toolchain).
Documenting all those possibilities would go far beyond the scope of `v4l2loopback`.
Please understand that we cannot provide support for questions regarding dependencies.


# BUILD
To build the kernel module, run:

    $ make

This should give you a file named "v4l2loopback.ko", which is the kernel module

## Build again
You cannot load a module built for a specific version of the kernel into another version of the kernel.
So, if you have successfully built the module previously and have updated your kernel (and the matching headers)
In the meantime, you really must clean the build before re-compiling the module.
So run this *before* starting the build again:

    $ make clean

Afterwards re-run `make` to do the actual build.

## Build for a different kernel
By default a simple `make` will (try to) build the module for the currently active kernel (as determined by `uname -r`).
If you want to build for a different kernel, youcan do so by providing the kernel version via the `KERNELRELEASE` variable:

    $ make KERNELRELEASE=6.11.7-amd64

(Of course you must have the kernel-headers for the specified kernel available in the `/lib/modules/${KERNELRELEASE}/build/` directory.)


# INSTALL
To install the module, run "make install" (you might have to be 'root' to have
all necessary permissions to install the module).

If your system has "sudo", do:

    $ make && sudo make install
    $ sudo depmod -a

If your system lacks "sudo", do:

    $ make
    $ su
    (enter root password)
    # make install
    # depmod -a
    # exit


(The `depmod -a` call will re-calculate module dependencies, in order to
automatically load additional kernel modules required by v4l2loopback.
The call may not be necessary on modern systems.)

See below for [distribution-specific build instructions](#DISTRIBUTIONS)
or when using frameworks like [`DKMS`](#DKMS).


# RUN
Load the v4l2loopback module as root :

    # modprobe v4l2loopback

Using `sudo` use:

    $ sudo modprobe v4l2loopback

You can check which loopback devices are created by listing contents of
`/sys/devices/virtual/video4linux` directory. E.g. if there are two
`v4l2loopback` devices `/dev/video0` and `/dev/video3` you would get:

    $ ls -1 /sys/devices/virtual/video4linux
    video0
    video3

These devices are ready to accept contents to show.

Tested feeders:
- GStreamer-1.0: using the  "v4l2sink" element
- Gem(>=0.93) using the "recordV4L2" plugin

In theory most programs capable of _writing to_ a v4l2 device should work.

The data sent to the v4l2loopback device can then be read by any v4l2-capable
application.

You can find a number of scenarios on the wiki at
	http://github.com/umlaeute/v4l2loopback/wiki

## Troubleshooting
If you have a secure-boot enabled kernel, you might not be able to simply build a kernel module and insert it.
(You will get **SSL error**s when building the module.)
This is actually a security feature (as it prevents malicious code to be inserted into kernel-space).

If you are not allowed to insert the kernel module (running `modprobe`, or `insmod`), you have a few options
(consult your distribution's documentation on how to perform any of these steps):
- disable secure-boot and reboot
- sign the module binary with a whitelisted key (this probably only applies if you are creating a distribution)

You could also just try building the module [via `DKMS`](#DKMS), and hope that it does all the magic for you.

# OPTIONS
If you need several independent loopback devices, you can pass the "devices"
option, when loading the module; e.g.

    # modprobe v4l2loopback devices=4

Will give you 4 loopback devices (e.g. `/dev/video1` ... `/dev/video5`)

You can also specify the device IDs manually; e.g.

    # modprobe v4l2loopback video_nr=3,4,7

Will create 3 devices (`/dev/video3`, `/dev/video4` & `/dev/video7`)

    # modprobe v4l2loopback video_nr=3,4,7 card_label="device number 3","the number four","the last one"

Will create 3 devices with the card names passed as the second parameter:
- `/dev/video3` -> *device number 3*
- `/dev/video4` -> *the number four*
- `/dev/video7` -> *the last one*


If you encounter problems detecting your device with Chrome/WebRTC you can try 'exclusive_caps' mode:

    # modprobe v4l2loopback exclusive_caps=1

This will enable 'exclusive_caps' mode that only reports CAPTURE/OUTPUT capabilities exclusively.
The newly created device will announce OUTPUT capabilities only (so ordinary webcam applications
(including Chrome) won't see it). As soon as you have attached a producer to the device, it will
start announcing CAPTURE capabilities only (so applications that refuse to open devices that have
other capabilities apart from capturing can open it too.)

## CHANGING OPTIONS
Options that you provided when loading the module (e.g. via `modprobe`) cannot be easily changed
on the fly.
In order to change these options, you must first unload the module with `rmmod`
(which will only work if no application is any longer accessing one of the loopback devices)
and then load it again (with the new options).

See also the section about [DYNAMIC DEVICE MANAGEMENT](#dynamic-device-management).



# ATTRIBUTES
you can set and/or query some per-device attributes via sysfs, in a human
readable format. See `/sys/devices/virtual/video4linux/video*/`

also there are some V4L2 controls that you can list with

    $ v4l2-ctl -d /dev/video0 -l

- `keep_format(0/1)`: while set to 1, once negotiated format will be fixed forever,
                  until the setting is set back to 0
- `sustain_framerate(0/1)`: if set to 1, nominal device fps will be ensured by means
                        of frame duplication when needed
- `timeout(integer)`: if >0, will cause a timeout picture (a null frame, by default)
                  to be displayed after (value) msecs of missing input
- `timeout_image_io(0/1)`: if set to 1, the next opener will write to timeout frame
                       buffer

# CHANGING THE RUNTIME BEHAVIOUR
## FORCING FPS

    $ v4l2loopback-ctl set-fps /dev/video0 25

or

    $ echo '@100' | sudo tee /sys/devices/virtual/video4linux/video0/format

## FORCING FORMAT

    $ v4l2loopback-ctl set-caps /dev/video0 "UYVY:640x480"

Please note that *GStreamer-style caps* (e.g. `video/x-raw,format=UYVY,width=640,height=480`) are no longer supported!

## SETTING STREAM TIMEOUT

You can define a timeout (in milliseconds), after which the loopback device will start outputting NULL frames,
if the producer suddenly stopped.

~~~
$ v4l2-ctl -d /dev/video0 -c timeout=3000
~~~

_Requires GStreamer 1.0, version >= 1.16_: You can provide a timeout image,
which will be displayed (instead of the NULL frames), if the
producer doesn't send any new frames for a given period (in the following
example; 3000ms):

~~~
$ v4l2loopback-ctl set-timeout-image -t 3000 /dev/video0 service-unavailable.png
~~~

## DYNAMIC DEVICE MANAGEMENT
You can create (and delete) loopback devices on the fly, using the `add` (resp. `delete`) commands of the `v4l2loopback-ctl` utility.

When creating a new device, module options might be ignored. So you must specify them explicitly.

To create a new device `/dev/video7` that has a label "loopy doopy", use:

~~~
$ sudo v4l2loopback-ctl add -n "loopy doopy" /dev/video7
~~~

Deleting devices is as simple as:

~~~
$ sudo v4l2loopback-ctl delete /dev/video7
~~~

# KERNELs
The original module has been developed for linux-2.6.28;
I don't have a system with such an old kernel anymore, so I don't know whether
it still works.
Further development has been done mainly on linux-2.6.32 and linux-2.6.35, with
newer kernels being continually tested as they enter Debian.

Support:
- >= <kbd>5.0.0</kbd>		should work
- >= <kbd>4.0.0</kbd>		should work
- >= <kbd>3.0.0</kbd>		might work
- << <kbd>3.0.0</kbd>		may work (has not been tested in ages)
- <= <kbd>2.6.37</kbd>		will definitely NOT work

# DISTRIBUTIONS
v4l2loopack is now (since 2010-10-13) available as a Debian-package.
https://packages.debian.org/source/stable/v4l2loopback

This means, that it is also part of Debian-derived distributions, including
Ubuntu (starting with natty).
The most convenient way is to install the package "v4l2loopback-dkms":

    # apt-get install v4l2loopback-dkms

This should automatically build and install the module for your current kernel
(provided you have the matching kernel-headers installed).
Another option is to install the "v4l2loopback-source" package.
In this case you should be able to simply do (as root):

    # apt-get install v4l2loopback-source module-assistant
    # module-assistant auto-install v4l2loopback-source

# DKMS
The *Dynamic Kernel Module Support framework* (DKMS) is designed to allow
individual kernel modules to be upgraded without changing the whole kernel.
It is also very easy to rebuild modules as you upgrade kernels.

If your distribution doesn't provide `v4l2loopback`-packages (or they are too old)
and you are experiencing troubles with code-signing, you probably should try this.

E.g. to build the v4l2loopback-v0.12.5 (but check the webpage for newer releases first!),
use something like the following (you might need to run the `dkms` commands as superuser/root):

~~~
version=0.12.5
# download and extract the tarball (tar requires superuser privileges)
curl -L https://github.com/umlaeute/v4l2loopback/archive/v${version}.tar.gz | tar xvz -C /usr/src
# build and install the DKMS-module (requires superuser privileges)
dkms add -m v4l2loopback -v ${version}
dkms build -m v4l2loopback -v ${version}
dkms install -m v4l2loopback -v ${version}
~~~~

| distribution       | dependencies          |
|--------------------|-----------------------|
| Fedora,...         | gcc kernel-devel dkms |
| Debian, Ubuntu,... | dkms                  |

_Note_: Using this method will _NOT_ install the `v4l2loopback-ctl` tool; do so manually via
`cd utils && make && sudo make install`.

# LOAD THE MODULE AT BOOT

One can avoid manually loading the module by letting systemd load the module
at boot, by creating a file `/etc/modules-load.d/v4l2loopback.conf` with just
the name of the module. This is especially convenient when `v4l2loopback` is installed with DKMS or with
a package provided by your Linux distribution:

~~~
v4l2loopback
~~~

If needed, one can specify default module options by creating
`/etc/modprobe.d/v4l2loopback.conf` in the following form instead:

~~~
options v4l2loopback video_nr=3,4,7 card_label="device number 3,the number four,the last one"
~~~

These options also become the defaults when manually calling
`modprobe v4l2loopback`. Note that the double quotes can only be used at the
beginning and the end of the option's value, as opposed to when they are
specified on the command line.

If your system boots with an initial ramdisk, which is the case for most
modern distributions, you need to update this ramdisk with the settings above,
before they take effect at boot time. In Ubuntu, this image is updated with
`sudo update-initramfs`. The equivalent on Fedora is `sudo dracut -f`.


# DOWNLOAD
The most up-to-date version of this module can be found at
http://github.com/umlaeute/v4l2loopback/.

# LICENSE/COPYING

- Copyright (c) 2010-2023 IOhannes m zmoelnig
- Copyright (c) 2016 Gavin Qiu
- Copyright (c) 2016 George Chriss
- Copyright (c) 2014-2015 Tasos Sahanidis
- Copyright (c) 2012-2015 Yusuke Ohshima
- Copyright (c) 2015 Kurt Kiefer
- Copyright (c) 2015 Michel Promonet
- Copyright (c) 2015 Paul Brook
- Copyright (c) 2015 Tom Zerucha
- Copyright (c) 2013 Aidan Thornton
- Copyright (c) 2013 Anatolij Gustschin
- Copyright (c) 2012 Ted Mielczarek
- Copyright (c) 2012 Anton Novikov
- Copyright (c) 2011 Stefan Diewald
- Copyright (c) 2010 Scott Maines
- Copyright (c) 2009 Gorinich Zmey
- Copyright (c) 2005-2009 Vasily Levin

    This package is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 2 of the License, or
    (at your option) any later version.

    This package is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program. If not, see <http://www.gnu.org/licenses/>.
