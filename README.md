# Futro-S740-Homeserver
Notes on how to set up a fully virtualized homeserver incl. local HTPC on the Fujitsu Futro S740 with Proxmox

## Basics and Hardware Information
See https://github.com/R3NE07/Futro-S740

## Operating system (Proxmox) and iGPU passthrough
I used [this guide](https://3os.org/infrastructure/proxmox/gpu-passthrough/igpu-passthrough-to-vm/#linux-virtual-machine-igpu-passthrough-configuration) to setup the host, but slightly adapted the guest hardware settings.

Still working on an up-to-date version of Proxmox to allow iGPU passthrough for the Gemini Lake UHD Graphics 600. My settings worked with this version:
**Proxmox 7.4-3 with Kernel 5.11.22-7-pve**
(5.13.19-1, 6.2.16-19-pve (Proxmox 8.0.4, Dani) and 6.5.11-7-pve might be worth to test next, discussion on supported versions is [here](https://forum.proxmox.com/threads/pci-passthrough-error-since-kernel-5-13-19-1-upgrade-from-7-0-to-7-1.100961/page-3))

1) Download Proxmox at https://www.proxmox.com/de/downloads/proxmox-virtual-environment/iso/proxmox-ve-7-4-iso-installer , install vioa USB stick and boot
2) Connect via webinterface at [IP]:8006, login with root and the password set during the installation process and enter the console by clicking on the node's name on the left
3) Find available kernels with `pve-efiboot-tool kernel list`/`proxmox-boot-tool kernel list` (seem to be identical and only showing the currently installed kernels) and `apt list pve-kernel*`
4) Install the newest working kernel (to be checked if newer kernels will support this, but currently this is the newest that I found): `apt install pve-kernel-5.11.22-7-pve `
5) Edit the cmdline in Grub `nano /etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet nowatchdog ipv6.disable=1 nofb nomodeset disable_vga=1 intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 modprobe.blacklist=radeon,nouveau,nvidia,nvidiafb,nvidia-gpu,snd_hda_intel,snd_soc_skl,snd_soc_avs,snd_sof_pci_intel_apl,snd_hda_codec_hdmi,i915 vfio-pci.ids=8086:3185,8086:3198"
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```
6) `update-grub`
7) `nano /etc/modules`
```
# Modules required for PCI passthrough
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
8) `nano /etc/modprobe.d/kvm.conf`
```
options kvm ignore_msrs=1
options kvm report_ignored_msrs=0
```
9) `nano /etc/modprobe.d/pve-blacklist.conf` (Blacklisting bluetooth here for power saving purposes. The rest is relevant to free the devices on the host system and make them available to the VM guest)
```
# nidiafb see bugreport https://bugzilla.proxmox.com/show_bug.cgi?id=701
blacklist nvidiafb
blacklist snd_hda_intel
blacklist snd_hda_codec_hdmi
blacklist i915
blacklist snd_soc_skl
blacklist snd_sof_pci
blacklist bluetooth
```
10) `update-initramfs -u -k all`
11) Another `update-grub` might be needed
12) Reboot the Proxmox host (setup SSH server before if you wish to have handy access next to NOVNC)
13) [Resize local and local-lvm storage](https://www.reddit.com/r/Proxmox/comments/vj6u54/is_it_possible_to_shrink_storage_disk_i_want_to/) to not waste storage for the root partition, I used on a 256GB SSD (238,98 effectively):
    - 8GB Swap
    - 30GB Root
    - 200GB local-lvm (30GB infra VM, min. 150GB media VM)

## Docker host or basic VM guest with access to iGPU

### VM Setup
1) On the host, find the iGPU: `lspci -nnv | grep VGA` (should be 0000:00:02 on the Futro)
2) Setup the following VM:

| Attribute  | Setting |
| ------------- | ------------- |
| Memory  | 2.00  |
| Processors  | 3 [host,flags=+pdpe1gb;+aes][cpuunits=100]  |
| BIOS  | SeaBIOS  |
| Display  | VirtIO-GPU (virtio)  |
| Machine  | Default (i440fx)  (q35 proved to work as well, but I did not manage to have the virtual console usable due to the qemu soundchip not being recognized when passing through the host sound chip)|
| SCSI Controller  | VirtIO SCSI single  |
| Hard Disk  | local-lvm:vm-102-disk-0,iothread=1,size=20G  |
| Network Device (net0)  | virtio=...,bridge=vmbr0,firewall=1  |
| USB Device (usb0)  | host=046d:c512 (USB mouse and keyboard directly attached to the S740) |
| USB Device (usb1)  | host=045e:07b2 (USB mouse and keyboard directly attached to the S740) |
| PCI Device (hostpci0)  | **0000:00:02 (all functions!), rombar=0** (the graphics card) |
| PCI Device (hostpci1)  | **0000:00:0e, (all functions!) rombar=0** (the audio chip) |

**Note:** Audio Passthrough is [reported](https://www.mydealz.de/comments/permalink/38190848) to only function with OVMF BIOS but I got audio output on i440fx/SeaBIOS through the front audio jack with `sudo speaker-test -D plughw:1,0` (hostpci1 with All functions, ROM-Bar and PCIe checked on a q35 VM)

Working on a solution with OVMF but did not succeed yet. [Thread on Proxmox forum](https://forum.proxmox.com/threads/intel-igp-gemini-lake-passthrough-q35-fails-to-boot-on-ubuntu-18-04-3-lts-%E2%80%93-i915-conflict-detected-with-stolen-region.57584/) regarding Ubuntu guest on Gemini Lake with q35 as well as [this one](https://forum.proxmox.com/threads/proxmox-6-0-gemini-lake-and-igd-graphics-passthrough-for-windows-10.60415/page-3#post-389588) might help

3) Install e.g. Debian in the guest VM. Ideally, unselect the desktop environment and only install SSH server and the basic sytem utilities. In case of using a desktop environment, make sure to make the physical display your main display. Then you can basically use the connected USB mouse and keyboard as if your are working with a native system. You can even disable the NOVNC screen, yet I found it helpful to use this screen for the NOVNC terminal in runlevel 3 while having all graphical outputs (KODI, UxPlay) on the physical display.
5) Optional: Enable xterm.js by adding a virtual serial port to the VM, enable the serial port in the VM operating system `sudo systemctl enable serial-getty@ttyS0.service` and `sudo systemctl start serial-getty@ttyS0.service`.
6) Install and set up ssh
7) Install and set up unattended-upgrades
8) Install and set up fail2ban

### Docker setup

1) `sudo apt install docker.io`
2) `sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart-always -v /var/run/docker.sock:/var/run/docker.sock -v /root/containers/portainer:/data portainer/portainer-ce:latest`

### Local HTPC Outputs

#### UxPlay (Airplay Server)
For details see https://github.com/FDH2/UxPlay

Uxplay is part of a standard Debian distribution. For me it runs either on a separate VM only used for local HTPC output, ie. iGPU, or together with the Docker host. Benefit of having all together is that also Docker containers can use hardware transcoding, e.g. Jellyfin!

Install GStreamer dependencies and UxPlay itself (installs >300 dependencies as well :( )
```sudo apt install gstreamer1.0-plugins-base gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-bad uxplay```

Uxplay needs to be started with manual selection of video and audio sinks for the Futro S740 (in my case the framebuffer device for DisplayPort 1 while running the VirtIO screen / NOVNC on /dev/fb0):
```
uxplay -n Homeserver -nh -s 1280x1024 -nohold -vs "fbdevsink device=/dev/fb1"
```

Note/Challenges:
- At least with a q35 VM, Uxplay does not work when run as root and seems not to initialize the server socket(s) (might become a challenge when UxPlay is supposed to run in a docker container under root)


#### Kodi
Kodi can be installed and started on my system without desktop environment. Will install a >110 depenendencies though (with UxPlay and Gstreamer already installed, thus on-top of their dependencies):
`apt install kodi`
If Kodi is run as an unpriviledged user (recommended!) make sure to add the user to the group `input` (`usermod -a -G input administrator`) and reboot the VM to be able to use mouse and keyboard!

Note/Challenges:
- Running in parallel to UxPlay on a single VM seemed to cause Kodi to overlay the UxPlay image (at least it was not visible but Kodi kept in the foreground independently of the sink used for UxPlay)
- Activating the webserver in guisettings.xml to be able to steer Kodi via mobile app is always reset at start
- How to run Kodi? Docker container (no working container known which directly accesses the framebuffer without the need for a GUI environment) vs. native in a VM together with UxPlay?
