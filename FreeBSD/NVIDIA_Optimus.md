NVIDIA Optimus on FreeBSD
=========================

This is a little collection about what you can and cannot do with
Optimus on FreeBSD.

If you are looking for a *working* solution, skip ahead to the
[VirtualGL section](#using-virtualgl).

Using the Intel GPU as an Output Provider
-----------------------------------------

The recommended way is to use Optimus via Xorg is to use
`xrandr --setprovideroutputsource`. This requires using the intel
device as a `GPUDevice` and sinking the output into it:

```sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
```

Setting up Xorg with the NVIDIA card as screen 0, the intel device
is automatically recognised as a GPUDevice. Unfortunately then Xorg
unloads the driver, because the device doesn't have a dedicated screen:

`/var/log/Xorg.0.log`
```xorg.conf
…
[154047.885] (**) ServerLayout "layout_nvidia"
[154047.885] (**) |-->Screen "screen_nvidia" (0)
[154047.885] (**) |   |-->Monitor "monitor_DP0"
[154047.885] (**) |   |-->Device "device_nvidia"
[154047.885] (**) |   |-->GPUDevice "device_intel"
…
[154047.925] (**) modeset(1): claimed PCI slot 0@0:2:0
[154047.925] (II) modeset(1): using default device
[154047.925] (WW) VGA arbiter: cannot open kernel arbiter, no multi-card support
[154047.925] (EE) Screen 1 deleted because of no matching config section.
[154047.925] (II) UnloadModule: "modesetting"
…
```

As a result the intel card is not available as an output:

```
# xrandr --listproviders
Providers: number : 1
Provider 0: id: 0x1b8 cap: 0x0 crtcs: 4 outputs: 8 associated providers: 0 name:NVIDIA-0
# xrandr --listproviders -display :0.1
Can't open display :0.1
#
```

Giving it a screen, fails because a device cannot host a screen and
be a GPUDevice:

`/var/log/Xorg.0.log`
```xorg.conf
…
[154553.760] (**) ServerLayout "layout_nvidia"
[154553.760] (**) |-->Screen "screen_nvidia" (0)
[154553.760] (**) |   |-->Monitor "monitor_DP0"
[154553.760] (**) |   |-->Device "device_nvidia"
[154553.760] (**) |   |-->GPUDevice "device_intel"
[154553.760] (**) |-->Screen "screen_intel" (1)
[154553.760] (**) |   |-->Monitor "monitor_eDP1"
[154553.760] (**) |   |-->Device "device_intel"
…
[154553.800] (**) modeset(1): claimed PCI slot 0@0:2:0
[154553.800] (II) modeset(1): using default device
[154553.800] (WW) VGA arbiter: cannot open kernel arbiter, no multi-card support
[154553.800] (EE) Screen 1 deleted because of no matching config section.
[154553.800] (II) UnloadModule: "modesetting"
…
```

Only using the intel device as a dedicated screen host works:

`/var/log/Xorg.0.log`
```xorg.conf
…
[154677.307] (**) ServerLayout "layout_nvidia"
[154677.307] (**) |-->Screen "screen_nvidia" (0)
[154677.307] (**) |   |-->Monitor "monitor_DP0"
[154677.307] (**) |   |-->Device "device_nvidia"
[154677.307] (**) |-->Screen "screen_intel" (1)
[154677.307] (**) |   |-->Monitor "monitor_eDP1"
[154677.307] (**) |   |-->Device "device_intel"
…
[154677.347] (**) modeset(1): claimed PCI slot 0@0:2:0
[154677.347] (II) modeset(1): using default device
[154677.347] (WW) VGA arbiter: cannot open kernel arbiter, no multi-card support
[154677.347] (II) NVIDIA(0): Creating default Display subsection in Screen section
…
```

Unfortunately this means it is not available as a GPUDevice:

```
# xrandr --listproviders
Providers: number : 1
Provider 0: id: 0x1b8 cap: 0x0 crtcs: 4 outputs: 8 associated providers: 0 name:NVIDIA-0
# xrandr --listproviders -display :0.1
Providers: number : 1
Provider 0: id: 0x1ea cap: 0xa, Sink Output, Sink Offload crtcs: 3 outputs: 1 associated providers: 0 name:modes
etting
# xrandr --setprovideroutputsource modesetting NVIDIA-0
Could not find provider with name modesetting
# xrandr --setprovideroutputsource modesetting NVIDIA-0 -display :0.1
Could not find provider with name NVIDIA-0
#
```

So currently this way is not viable.

Using VirtualGL
---------------

VirtualGL is 3D rendering offloading solution for thin clients, which
can also be used to run 3D accelerated software on one GPU and display
it on another. Its overhead is considerable, so your mileage may vary.

There are two ways to set this up, creating a second headless Xorg
instance for the NVIDIA card or hosting it on a second screen within
the user facing Xorg session.

The first variant has the advantage of using the intel and NVIDIA 3D
acceleration, but it's a pain to set up. So the single Xorg instance
variant is presented here.

### Getting a Working `xorg.conf`

First of all the PCI bus IDs of both cards are required:

```
# pciconf -lv | grep -A2 vga
vgapci1@pci0:0:2:0:     class=0x030000 card=0x65d11558 chip=0x3e9b8086 rev=0x00 hdr=0x00
    vendor     = 'Intel Corporation'
    device     = 'UHD Graphics 630 (Mobile)'
--
vgapci0@pci0:1:0:0:     class=0x030000 card=0x65d11558 chip=0x1f1010de rev=0xa1 hdr=0x00
    vendor     = 'NVIDIA Corporation'
    device     = 'TU106M [GeForce RTX 2070 Mobile]'
#
```

This translates into the following configuration:

`/usr/local/etc/X11/xorg.conf.d/video.conf`
```xorg.conf
Section "ServerLayout"
	Identifier     "layout_intel"
	Screen         0 "screen_intel"
	Screen         1 "screen_nvidia"
EndSection

Section "Screen"
	Identifier     "screen_intel"
	Device         "device_intel"
EndSection

Section "Screen"
	Identifier     "screen_nvidia"
	Device         "device_nvidia"
EndSection

Section "Device"
	Identifier     "device_intel"
	Driver         "modesetting"
	BusID          "PCI:0:2:0"
EndSection

Section "Device"
	Identifier     "device_nvidia"
	Driver         "nvidia"
	BusID          "PCI:1:0:0"
	Option         "AllowEmptyInitialConfiguration" "true"
EndSection
```

The `AllowEmptyInitialConfiguration` option prevents the NVIDIA driver
from failing without a connected monitor.

Once logged into X both screens should be verified bu running xrandr:

```
# xrandr
Screen 0: minimum 320 x 200, current 3840 x 2160, maximum 8192 x 8192
eDP-1 connected primary 3840x2160+0+0 (normal left inverted right x axis y axis) 344mm x 194mm
   3840x2160     60.00*+  59.98    59.97  
…
# xrandr -display :0.1
Screen 1: minimum 8 x 8, current 640 x 480, maximum 32767 x 32767
DP-0 disconnected primary (normal left inverted right x axis y axis)
DP-1 disconnected (normal left inverted right x axis y axis)
DP-2 disconnected (normal left inverted right x axis y axis)
DP-3 disconnected (normal left inverted right x axis y axis)
HDMI-0 disconnected (normal left inverted right x axis y axis)
DP-4 disconnected (normal left inverted right x axis y axis)
DP-5 disconnected (normal left inverted right x axis y axis)
DP-6 disconnected (normal left inverted right x axis y axis)
#
```

If setting up the NVIDIA device failed, the output would look like this:

```
# xrandr -display :0.1
Can't open display :0.1
#
```

In that case check the `/var/log/Xorg.0.log` file to figure out what
went wrong. Otherwise proceed with setting up VirtualGL.

### Setting up VirtualGL

First install the
[virtualgl package](https://www.freshports.org/x11/virtualgl/).

Create a small script:

`~/bin/optirun`
```sh
#!/bin/sh
export PATH="/usr/local/VirtualGL/bin:$PATH"
exec /usr/local/VirtualGL/bin/vglrun -d :0.1 "$@"
```

Make the script executable and test it:

```sh
# chmod +x ~/bin/optirun
# optirun glxinfo|grep direct
direct rendering: Yes
…
# optirun glxgears
505 frames in 5.0 seconds = 100.988 FPS
616 frames in 5.0 seconds = 123.052 FPS
695 frames in 5.0 seconds = 138.820 FPS
697 frames in 5.0 seconds = 139.243 FPS
696 frames in 5.0 seconds = 139.038 FPS
#
```

### Using Intel HW Acceleration

If for some reason the intel acceleration is desired the following should
be added to the Xorg configuration:

```xorg.conf
Section "Files"
	ModulePath     "/usr/local/lib/xorg/modules/extensions/.xorg"
	ModulePath     "/usr/local/lib/xorg/modules"
EndSection
```

This gives the Xorg libglx module precedence over the NVIDIA provided
version.

And the NVIDIA libGL needs to be deactivated:

```
# mv /usr/local/etc/libmap.d/nvidia.conf /usr/local/etc/libmap.d/nvidia.conf_
#
```

After a restart of Xorg 3D applications should work without prefixing
the optirun command.

Note, keep the NVIDIA screen around even when not intending to use
Optimus. The NVIDIA hardware draws less power when the driver is active.
