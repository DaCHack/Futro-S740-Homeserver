# Futro-S740-Homeserver
Notes on how to set up a fully virtualized homeserver incl. local HTPC on the Fujitsu Futro S740 with Proxmox

## Basics and Hardware Information
See https://github.com/R3NE07/Futro-S740

## Operating system (Proxmox) and iGPU passthrough
I used [this guide](https://3os.org/infrastructure/proxmox/gpu-passthrough/igpu-passthrough-to-vm/#linux-virtual-machine-igpu-passthrough-configuration) to setup the host, but slightly adapted the guest hardware settings.

Regarding swap settings on the hypervisor: Recommendation is to [not use swap in Proxmox](https://forum.proxmox.com/threads/what-is-the-better-solution-swapping-on-host-or-swapping-in-the-vm.34740/), but ensure that the VMs have enough swap space that they manage themselves and if so, only do this to cover potential over-provisioning of VM RAM.

**Compatibility list**

| Proxmox Version  | Kernel Version | Video | Audio | Other comments
| ------------- | ------------- | ------------- | ------------- | ------------- |
| 7.4-3  | 5.11.22-7-pve  | OK | OK | First complete successful runthough with i440fx/Seabios |
| 7.4-3  | 5.13.19-1      | - | - | Untested, [supposed to work](https://forum.proxmox.com/threads/pci-passthrough-error-since-kernel-5-13-19-1-upgrade-from-7-0-to-7-1.100961/page-3) |
| 8.0.4  | 6.2.16-19-pve  | - | - | Untested, what Dani uses with Audio+Video on q35/Seabios |
| 8.0.4 (?) | 6.5.11-7-pve   | - | - | Untested
| 8.2.2  | 6.8.4-2-pve <br/>6.8.12-2-pve    | OK | (OK) | - Currently testing with q35/Seabios, audio output only with `snd_hda_intel.probe_mask=1` in cmdline. KODI runs fine, no audio output from UxPlay until I installed, ran and then closed again KODI<br>- Currently testing with q35/OVMF working quite well with only the virtual console being unusable. SSH required|

1) Download Proxmox at [Version 7.4](https://www.proxmox.com/de/downloads/proxmox-virtual-environment/iso/proxmox-ve-7-4-iso-installer) or [current](https://www.proxmox.com/de/downloads/proxmox-virtual-environment/iso), install via USB stick and boot
   - On installation target page go to "Options" and set swapsize and maxroot. To avoid changing it later manually (see step 14 on what I used)
2) Connect via webinterface at [IP]:8006, login with root and the password set during the installation process and enter the console by clicking on the node's name on the left
3) Find available kernels with `pve-efiboot-tool kernel list`/`proxmox-boot-tool kernel list` (seem to be identical and only showing the currently installed kernels) and `apt list pve-kernel*` or `apt list proxmox-kernel*` respectively
4) Install the newest working kernel (as listed above) if the stock kernel does not work: `apt install pve-kernel-5.11.22-7-pve `
5) Edit the cmdline in Grub `nano /etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet nowatchdog nofb nomodeset disable_vga=1 intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 modprobe.blacklist=radeon,nouveau,nvidia,nvidiafb,nvidia-gpu,snd_hda_intel,snd_soc_skl,snd_soc_avs,snd_sof_pci_intel_apl,snd_hda_codec_hdmi,i915 vfio-pci.ids=8086:3185,8086:3198"
GRUB_CMDLINE_LINUX=""
```
I also tried to avoid the security-sensitive ACS-Overrides [see here](https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html) and [here](https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/) which seemed to work fine a VM already set up before. To be tested on a newly created machine in an empty Proxmox:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet nowatchdog nofb nomodeset disable_vga=1 intel_iommu=on iommu=pt initcall_blacklist=sysfb_init video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 modprobe.blacklist=radeon,nouveau,nvidia,nvidiafb,nvidia-gpu,snd_hda_intel,snd_soc_skl,snd_soc_avs,snd_sof_pci_intel_apl,snd_hda_codec_hdmi,i915 vfio-pci.ids=8086:3185,8086:3198"
GRUB_CMDLINE_LINUX=""
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
12) Reboot the Proxmox host (setup SSH server before if you wish to have handy access next to NOVNC - should already be running by default)
13) Set up HTTPS certificates via Let's encrypt in [Node]->Certificates and Datacenter->ACME. For IONOS I needed to append in "API Data":
```
IONOS_PREFIX = <XXXXXXX>
IONOS_SECRET = <XXXXXXX>
```

14) [Resize local and local-lvm storage](https://www.reddit.com/r/Proxmox/comments/vj6u54/is_it_possible_to_shrink_storage_disk_i_want_to/) to not waste storage for the root partition, if not done during installation (see above). I used on a 256GB SSD (238,98 effectively):
    - 8GB Swap
    - 30GB Root
    - 200GB local-lvm (30GB infra VM, min. 150GB media VM)
   
I noticed that local-lvm was sized smaller than the available space after subtracting swap and root. So I extended the volume manually after installation based on this [documentation](https://pve.proxmox.com/wiki/Resize_disks#3._Enlarge_the_filesystem(s)_in_the_partitions_on_the_virtual_disk):
```
pvresize /dev/sda3
lvresize --extents +100%FREE -resizefs /dev/pve/data
```
   
15) Install and set up unattended-upgrades
```
sudo apt install unattended-upgrades
```
Edit `sudo nano /etc/apt/apt.conf.d/50unattended-upgrades` and make sure following three lines are uncommented:
```
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
```
Turn on the service and do a dry run to test the configuration:
```
sudo systemctl start unattended-upgrades
sudo systemctl enable unattended-upgrades
sudo unattended-upgrades --dry-run --debug
```

16) Install and set up fail2ban
```
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Edit `sudo nano /etc/fail2ban/jail.local` to secure the SSH server. Defaults can be kept. Just make sure that the SSH section looks as follows and add the Proxmox section:
```
[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
enabled = true
filter  = sshd
port    = ssh
logpath = %(sshd_log)s
#backend = %(sshd_backend)s
backend = systemd

[proxmox]
enabled = true
port = https,http,8006
filter = proxmox
backend = systemd
```
Create the Proxmox configuration file for fail2ban in `sudo nano /etc/fail2ban/filter.d/proxmox.conf`:
```
[Definition]
failregex = pvedaemon\[.*authentication failure; rhost=<HOST> user=.* msg=.*
ignoreregex =
```

Restart the fail2ban daemon:
```
sudo service fail2ban restart
```

17) Disable IPv6 if not needed via `nano /etc/sysctl.d/disable-ipv6.conf`:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

## Docker host or basic VM guest with access to iGPU

### VM Setup
1) On the host, find the iGPU: `lspci -nnv | grep VGA` (should be 0000:00:02 on the Futro)
2) Setup the following VM:

| Attribute  | My Setting on 7.4-3 | My Setting on 8.2.2 |
| ------------- | ------------- | ------------- |
| Memory  | 2.00  | 4.00 [balloon=0]  |
| Processors  | 3 [host,flags=+pdpe1gb;+aes][cpuunits=100]  | 3 [host,flags=+pdpe1gb;+aes][cpuunits=100]  |
| BIOS  | SeaBIOS  | OVMF (UEFI)  |
| Display  | VirtIO-GPU (virtio)  | VirtIO-GPU (virtio) (or "none" to not use the anyways unusable virtual console, but then having the physical display as the main screen) |
| Machine  | Default (i440fx)  (q35 proved to work as well, but I did not manage to have the virtual console usable due to the qemu soundchip not being recognized when passing through the host sound chip)|  q35 (virtual console also not usable here) |
| SCSI Controller  | VirtIO SCSI single  | VirtIO SCSI single  |
| Hard Disk  | local-lvm:vm-102-disk-0,iothread=1,size=20G  | local-lvm:vm-102-disk-0,iothread=1,size=32G (partitioned automatically via Debian installer |
| Network Device (net0)  | virtio=...,bridge=vmbr0,firewall=1  | virtio=...,bridge=vmbr0,firewall=1  |
| USB Device (usb0)  | host=046d:c512 (USB mouse and keyboard directly attached to the S740) | host=046d:c512 (USB mouse and keyboard directly attached to the S740) |
| USB Device (usb1)  | host=045e:07b2 (USB mouse and keyboard directly attached to the S740) | host=045e:07b2 (USB mouse and keyboard directly attached to the S740) |
| PCI Device (hostpci0)  | **0000:00:02 (all functions!), rombar=0** (the graphics card) | **0000:00:02 (all functions!), rombar=0** (the graphics card) |
| PCI Device (hostpci1)  | **0000:00:0e, (all functions!) rombar=0** (the audio chip) | **0000:00:0e, (all functions!) rombar=0** (the audio chip) |

**Note:** Audio works fine with Proxmox 8.2.2 and Kernel 6.8.4-2-pve on a vanilla q35 machine with OVMF. Make sure both GPU and HDA are passed through with all functions on, while ROM-Bar and PCIe both deactivated! Audio Passthrough is [reported](https://www.mydealz.de/comments/permalink/38190848) to only function with OVMF BIOS but I got audio output on i440fx/SeaBIOS through the front audio jack with `sudo speaker-test -D plughw:1,0` (hostpci1 with All functions, ROM-Bar and PCIe checked on a q35 VM)

**Note:** It seems that DisplayPort does not work when plugged in only after boot. I tested booting without the DP connected and did not receive any output until I first connected the monitor end and then plugged out and in again the PC end of the cable. Afterwards, the system detects the DP cable again even after the monitor end is completely cut off power. It needs about 10-20sec though for the connection to be established. Might be a special situation since I use a DP->HDMI cable and an HDMI-splitter between the PC and the monitor.

3) Install e.g. Debian in the guest VM.
   - Ideally, unselect the desktop environment and only install SSH server and the standard system utilities.
   - In case of using a desktop environment, make sure to make the physical display your main display. Then you can basically use the connected USB mouse and keyboard as if your are working with a native system.
   - You can even disable the NOVNC screen, yet I found it helpful to use this screen for the NOVNC terminal in runlevel 3 while having all graphical outputs (KODI, UxPlay) on the physical display. (not feasible on q35 machine, virtual console freezes and no clean reboots/shutdowns are possible via SSH)
   - **Disabling the VirtIO-GPU in machine options avoids a deadlock on the virtual console and enables clean shutdowns and reboots**
4) Optional: Enable xterm.js by adding a virtual serial port to the VM, enable the serial port in the VM operating system `sudo systemctl enable serial-getty@ttyS0.service` and `sudo systemctl start serial-getty@ttyS0.service`.
5)  Enable sudo for your non-root user (where administator is the user name created during installation) and *reboot*:
```
su -
apt install sudo
usermod -a -G sudo administrator
```
6) Strip some unnecessary packages from the Debian installation:
```
sudo apt remove debian-faq doc-debian eject file iamerican ibritish ispell laptop-detect reportbug tasksel vim-common vim-tiny wamerican whois dictionaries-common emacsen-common fonts-droid-fallback ghostscript libcups2 libgs-common libgs10 libgs10-common libidn12 libijs-0.35 libjbig2dec0 libmagic-mgc libmagic1 libpaper-utils libpaper1 poppler-data python3-certifi python3-chardet python3-charset-normalizer python3-debian python3-debianbts python3-httplib2 python3-idna python3-pkg-resources python3-pycurl python3-pyparsing python3-pysimplesoap python3-requests python3-six python3-urllib3
sudo apt autoremove
```
7)  [For newer Kernels if speaker-test is unsuccessful](https://bugzilla.kernel.org/show_bug.cgi?id=208511), add `snd_hda_intel.probe_mask=1` or `snd_hda_intel.power_save_controller=0` to `sudo nano /etc/default/grub` cmdline and `sudo update-grub` to get sound from the audio jack. This might cause no output possible via DisplayPort though! **I did not need this.**
8) Install non-free firmware
```
sudo apt install firmware-misc-nonfree
```
9) I usually create another non-root user separately from the main one to provide backup data to be pulled from external systems. This avoids having credentials for lateral movement on the source system. Setting a password is not needed as I only access the users home directory through key-based SSH login. Also install some tools I need for the bot scripts:
```
sudo useradd -m bot
sudo apt install icu-devtools gpg
```

10) Set up SSH
   - After Debian installation is set up already and you are able to log in with the user created during installation
   - You may need to log into the guest system via SSH because the virtual console is not available due to PCI passthrough!
   - Upload the `authorized_keys` file into the users' .ssh folder
   - Edit the `sudo nano /etc/ssh/sshd_config` file:
```
KbdInteractiveAuthentication yes
UsePAM yes
PasswordAuthentication yes
AuthenticationMethods "publickey,password"
X11Forwarding no
PrintMotd no  
PrintLastLog no

AllowUsers administrator bot

Match User bot
        PasswordAuthentication no
        AuthenticationMethods "publickey"
```

11) Install and set up unattended-upgrades
```
sudo apt install unattended-upgrades
```
Edit `sudo nano /etc/apt/apt.conf.d/50unattended-upgrades` and make sure following three lines are uncommented:
```
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
```
Turn on the service and do a dry run to test the configuration:
```
sudo systemctl start unattended-upgrades
sudo systemctl enable unattended-upgrades
sudo unattended-upgrades --dry-run --debug
```

12) Install and set up fail2ban
```
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Edit `sudo nano /etc/fail2ban/jail.local` to secure the SSH server (ideally the only service running on the worker VM while everything else is containerized). Defaults can be kept. Just make sure that the SSH section looks as follows:
```
[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
enabled = true
filter  = sshd
port    = ssh
logpath = %(sshd_log)s
#backend = %(sshd_backend)s
backend = systemd**
```
Restart the fail2ban daemon:
```
sudo service fail2ban restart
```


### Docker setup

Install Docker and deploy [Portainer](https://www.portainer.io/). If needed, make sure to add the path to the SSL certificates as another volume:
```
sudo apt install docker.io
sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v /opt/appdata/portainer:/data portainer/portainer-ce:latest
```
Install cifs-utils in case you want to make CIFS shares available to your containers as in my case:
```
sudo apt install cifs-utils
```

### Local HTPC Outputs

#### UxPlay (Airplay Server)
For details see https://github.com/FDH2/UxPlay

Uxplay is part of a standard Debian distribution. For me it runs either on a separate VM only used for local HTPC output, ie. iGPU, or together with the Docker host. Benefit of having all together is that also Docker containers can use hardware transcoding, e.g. Jellyfin!

##### In Docker Container (recommended)
It runs natively very smooth and responsive. Yet it boiles the worker VM's operating system and installs >320 dependencies at >740MB disk space as well :( . Thus I created a docker container to at least keep the base system clean:

https://github.com/DaCHack/UxPlay-docker

It needs a local Avahi daemon on the docker host though so install it before spinning up the container:
```
sudo apt install avahi-daemon
```

I totally recommend this approach, despite hardware acceleration through /dev/dri does not work well. Checking on this...

##### In Worker VM Operating System

Install GStreamer dependencies and UxPlay itself
```
sudo apt install gstreamer1.0-plugins-base gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-alsa uxplay
```

If you want to use **gstreamer1.0-vaapi**, you need to add the user to the video and render groups `sudo usermod -a -G render administrator` and `sudo usermod -a -G video administrator`. Yet I noticed that UxPlay will not work with VAAPI installed: With VirtIO-GPU activated gstreamer will fail to initialize the device eventhough you only want the output on the physical display and with VirtIO-GPU deactivated it fails with error: `no such element factory "vaapipostproc"! `. So deinstalled it again falling back to software de-/encoding if I understand it correctly. Causes ~20% load on the 3 CPUs I assigned to the VM when playing a fullscreen video from my iPhone.
```
sudo apt install gstreamer1.0-plugins-base gstreamer1.0-libav gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-alsa gstreamer1.0-vaapi uxplay
```

On Debian 12 uxplay needs to be [updated to testing](https://unix.stackexchange.com/a/253866) / manually compiled to allow for at least version 1.69 (stock at 1.62) to enable the -dacp argument which is supposed to help an script handling uxplay and Kodi in parallel. Also, this version connects clients much faster!

Uxplay needs to be started with manual selection of video and audio sinks for the Futro S740 (in my case the framebuffer device for DisplayPort 1 while running the VirtIO screen / NOVNC on /dev/fb0 - **if VirtIO-GPU is off, use /dev/fb0!**):
```
uxplay -n Homeserver -nh -s 1280x1024 -nohold -vs "fbdevsink device=/dev/fb1" -as "alsasink device=plughw:1,0"
```
(for sound output via headphone jack - change device ID for alsasink to the correct DP port if needed!)

Note: UxPlay is [not designed to be run as root](https://github.com/FDH2/UxPlay/issues/330). A quick test seems to show that server sockets get initialized but the dns-sd "bonjour" service doesnt connect. Actually it does run fine under root, when the firewall is turned off. You may have an active firewall with some ports open for uxplay as a user, but not set up for root. A docker container thus should run as an unpriviledged user (recommended anyways!)


#### Kodi

##### In Docker Container
Kodi can be installed and started on my system without desktop environment. Yet it boiles the worker VM's operating system and installs >110 dependencies at >240MB disk space though (with UxPlay and Gstreamer already installed, thus on-top of their dependencies). Thus I created a docker container to at least keep the base system clean:

https://github.com/DaCHack/rpi-kodi

Performance is yet a little shaky, but working on it.

##### In Worker VM Operating System

Install through APT:
```
apt install kodi
kodi-standalone
```
If Kodi is run as an unpriviledged user (recommended!) make sure to add the user to the group `input` (`usermod -a -G input administrator`) and reboot the VM to be able to use mouse and keyboard!

**Note:** Running in parallel to UxPlay on a single VM seems to cause Kodi to overlay the UxPlay image (at least it was not visible but Kodi kept in the foreground independently of the sink used for UxPlay). Plan is to write a script watching for UxPlay's "-dacp" file. If the file appears (and thus a connection to UxPlay is established), the script shall pause the Kodi container or shut it down. Eventually both containers could be merged to run the script directly in the container.

## Further references for troubleshooting
https://forum.proxmox.com/threads/igpu-passthrough-intel-j4105-gemini-lake-uhd-graphics-600.139505/
https://forums.unraid.net/topic/124100-no-display-output-with-igpu-passthrough-windows-vm/
https://forum.proxmox.com/threads/proxmox-5-2-gemini-lake-and-igd-graphics-passthrough-for-ubuntu-18.47129/page-2
https://asokolsky.github.io/proxmox/pcie-passthrough.html
https://pve.proxmox.com/wiki/PCI(e)_Passthrough
https://forum.proxmox.com/threads/pci-passthrough-error-since-kernel-5-13-19-1-upgrade-from-7-0-to-7-1.100961/
