# Futro-S740-Homeserver
Notes on how to set up a fully virtualized homeserver incl. local HTPC on the Fujitsu Futro S740 with Proxmox

## Basics and Hardware Information
See https://github.com/R3NE07/Futro-S740

## Operating system (Proxmox) and iGPU passthrough
I used [this guide](https://3os.org/infrastructure/proxmox/gpu-passthrough/igpu-passthrough-to-vm/#linux-virtual-machine-igpu-passthrough-configuration) to setup the host, but slightly adapted the guest hardware settings.

Still working on an up-to-date version of Proxmox to allow iGPU passthrough for the Gemini Lake UHD Graphics 600. My settings worked with this version:
**Proxmox 7.4-3 with Kernel 5.11.22-7-pve**
(6.5.11-7-pve might be worth to test next, discussion on supported versions is [here](https://forum.proxmox.com/threads/pci-passthrough-error-since-kernel-5-13-19-1-upgrade-from-7-0-to-7-1.100961/page-3))

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

## Docker host or basic VM guest with access to iGPU

### VM Setup
1) On the host, find the iGPU: `lspci -nnv | grep VGA`
2) Setup the following VM:

| Attribute  | Setting |
| ------------- | ------------- |
| Memory  | 2.00  |
| Processors  | 3 [host,flags=+pdpe1gb;+aes][cpuunits=100]  |
| BIOS  | SeaBIOS  |
| Display  | VirtIO-GPU (virtio)  |
| Machine  | Default (i440fx)  |
| SCSI Controller  | VirtIO SCSI single  |
| Hard Disk  | local-lvm:vm-102-disk-0,iothread=1,size=20G  |
| Network Device (net0)  | virtio=...,bridge=vmbr0,firewall=1  |
| USB Device (usb0)  | host=046d:c512 (USB mouse and keyboard directly attached to the S740) |
| USB Device (usb1)  | host=045e:07b2 (USB mouse and keyboard directly attached to the S740) |
| PCI Device (hostpci0)  | **0000:00:02, rombar=0** (the graphics card) |
| PCI Device (hostpci1)  | **0000:00:0e, rombar=0** (the audio chip) |

**Note:** Audio Passthrough is [reported](https://www.mydealz.de/comments/permalink/38190848) to only function with OVMF BIOS but I got audio output through the front audio jack

Working on a solution with OVMF but did not succeed yet. [Thread on Proxmox forum](https://forum.proxmox.com/threads/intel-igp-gemini-lake-passthrough-q35-fails-to-boot-on-ubuntu-18-04-3-lts-%E2%80%93-i915-conflict-detected-with-stolen-region.57584/) regarding Ubuntu guest on Gemini Lake with q35 might help

3) Install e.g. Debian in the guest VM

## Local HTPC Outputs

### UxPlay (Airplay Server)
For details see https://github.com/FDH2/UxPlay

Uxplay is part of a standard Debian distribution. For me it runs either on a separate VM only used for local HTPC output, ie. iGPU, or together with the Docker host. Benefit of having all together is that also Docker containers can use hardware transcoding, e.g. Jellyfin!

Uxplay needs to be started with manual selection of video and audio sinks for the Futro S740 (in my case the framebuffer device for DisplayPort 1 while running the VirtIO screen / NOVNC on /dev/fb0):
```
uxplay -n Homeserver -vs "fbdevsink device=/dev/fb1"
```

## Kodi
Kodi can be installed and started on my system with KDE installed, but only booted into `sudo systemctl set-default multi-user.target`. So likely to work also with a Debian system without any desktop environment. X might need to be installed as a dependency though

Major challenges:
- Running in parallel to UxPlay caused kernel panics on my Raspberry Pi where I tried this setup before. To be checked how this can be overcome. Testing on both apps on a single VM seemed to cause Kodi to overlay the UxPlay image (at least it was not visible but Kodi kept in the foreground). Kodi does not react to mouse or keyboard input. Activating the webserver in guisettings.xml to be able to steer Kodi via mobile app is always reset at start
- How to run Kodi? Docker container (no working container known which directly accesses the framebuffer without the need for a GUI environment) vs. native in a VM together with UxPlay?
