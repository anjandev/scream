Scream - Virtual network sound card for Microsoft Windows
---------------------------------------------------------------
Scream is a virtual device driver for Windows that provides a
discrete sound device. Audio played through this device is
published on your local network as a PCM multicast stream.

Receivers on the network can pick up the stream and play it
through their own audio outputs. Two simple receivers for Linux
(interfacing with PulseAudio or ALSA) and one for Windows are
provided.

For the special scenario of a Windows guest on a QEMU host,
@martinellimarco has contributed support for transferring audio
via the IVSHMEM driver mechanism, similar to the GPU
pass-through software "Looking Glass". See the section on
IVSHMEM below.

Scream is based on Microsoft's MSVAD audio driver sample code.
The original code is licensed under MS-PL, as are my changes
and additions. See LICENSE for the actual license text.

Download and install
---------------------------------------------------------------
A ZIP file containing a signed x64 build is [available on the
GitHub releases page](https://github.com/duncanthrax/scream/releases).
The "installer" is a batch file that needs to be run with
administrator rights.

The build is supposed to run on all x64 versions of Windows 7
through Windows 10. 

Microsoft has [recently tightened the rules for signing kernel
drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/kernel-mode-code-signing-policy--windows-vista-and-later-). These new rules apply to newer Windows 10 installations
that were not upgraded from an earlier version. If your installation
is subject to these rules, the driver will not install.
**Workaround: [Disable secure boot in BIOS](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/disabling-secure-boot).**
For more information, see [this issue](https://github.com/duncanthrax/scream/issues/8).

Receivers
---------------------------------------------------------------
- Linux/Pulseaudio: Not included in the installer package. Just
type 'make' to build it. You might need to install build tools
and Pulseaudio development packages.

- Linux/ALSA: Originally contributed by @ivan. Not included in
the installer package. Just type 'make' to build it.You might
need to install build tools and ALSA development packages.

- Windows: ScreamReader, contributed by @MrShoenel. Included in
the installer package as of version 1.2. This receiver does not
support positional mapping of multichannel (more than stereo)
setups - meaning a mismatch in speaker setup can lead to channels
being played in the wrong position.

All three receivers can be run as unprivileged users. Receiver
systems that have an input firewall need to open UDP port 4010,
or whatever custom port you use.

Functional description
---------------------------------------------------------------
All audio played through the Scream device will be put onto
the local LAN as a multicast stream (using unicast is optional -
see below). Delay is minimal, since all processing is done
on kernel level. There is no userspace portion.

The multicast target address and port is "239.255.77.77:4010".
The audio is a raw PCM stream. The default sampling rate and
size can be set as the "Default format" in the driver "Advanced"
property page. The default speaker/channel configuration can be
set on the "Configure" dialog of the Scream sound device.

Data is transferred in UDP frames with a payload size of max.
1157 bytes, consisting of 5 bytes header and 1152 bytes PCM data.
The latter number is divisible by 4, 6 and 8, so a full number
of samples for all channels will always fit into a packet.
The first header byte denotes the sampling rate. Bit 7 specifies
the base rate: 0 for 48kHz, 1 for 44,1kHz. Other bits specify the
multiplier for the base rate. The second header byte denotes the
sampling width, in bits. The third header byte denotes the number
of channels being transferred. The fourth and fifth header bytes
make up the DWORD dwChannelMask from Microsofts WAVEFORMATEXTENSIBLE
structure, describing the mapping of channels to speaker positions.

Receivers simply need to read the stream off the network and
stuff it into a local audio sink. The receiver system's kernel
should automatically do the necessary IGMP signalling, so it's
usually sufficient to just open a multicast listen socket and
start reading from it. Minimal buffering (~ 4 times the UDP
payload size) should be done to account for jitter. 

Setting the sampling rate (optional)
-------------------------------------------------------------
To satisfy your audiophile feelings, or to reduce unnecessary
resampling, you might want to set a higher sampling rate and/or
sampling width. You can do that on the driver "Advanced" property
page, as shown below. *Warning:* using high sampling freqs with
24/32 bits width in combination with multichannel can lead to high
bit rates on the network. We recommend to stick to 48kHz/16 bits
for 5.1 or higher channel modes.

<img src="doc/sampling-rate.png" width="700"/>

Setting up default speaker configuration (optional)
-------------------------------------------------------------
Thanks to the great work of @martinellimarco, if your target
system has a multichannel speaker setup, you can extend that to
Windows as well. Use the "Configure" wizard of the sound device
driver dialog, as shown below. Please note that this is just a
system default, and that application software (like games) may
require their own settings to be made.

<img src="doc/speaker-setup.png" width="700"/>

Using unicast instead of multicast (optional)
-------------------------------------------------------------
This is only recommended if multicast gives you problems.
Tweak the registry in the manner depicted in this screenshot
(you will have to create the "Options" key), then reboot:

<img src="doc/registry.png"/>

Using IVSHMEM between Windows guest and Linux host
-------------------------------------------------------------
This can be used as an alternative to the default networked
transfer when using QEMU/KVM.
- Add a IVSHMEM device to your VM. We recommend a size of 2MB.
If you use other IVSHMEM devices, we recommend to use the same
domain and bus, just varying the slot numbers. Here is a config
example:
```
<shmem name='scream-ivshmem'>
  <model type='ivshmem-plain'/>
  <size unit='M'>2</size>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x11' function='0x0'/>
</shmem>
```
- Install the IVSHMEM driver from [here](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/).
- To make the driver use IVSHMEM, add a DWORD `HKLM\SYSTEM\CurrentControlSet\Services\Scream\Options\UseIVSHMEM`,
with the value being the size of the device in MB (2, as recommended). Please
note that scream will identify the device by its size, so you should only
have one IVSHMEM device with that specific size. Also note that you
might have to create the `Options` key. OR open the command prompt as administrator and run this command: `REG ADD HKLM\SYSTEM\CurrentControlSet\Services\Scream\Options /v UseIVSHMEM /t REG_DWORD /d 2`
- When the VM is running, check if the device exists as /dev/shm/scream-ivshmem,
and if the user you want to run the receiver as has read access.
If so, run a IVSHMEM-capable receiver with the path of the SHM file
as commandline parameter, for example:
```scream-ivshmem-pulse /dev/shm/scream-ivshmem```

Building
-------------------------------------------------------------
Visual Studio and a recent WDK are required. Good luck!
