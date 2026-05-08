# Dynamic GPU Passthrough to a Windows virtual machine using QEMU/libvirtd and Looking Glass with KVMFR
Scripts and file structure to setup dynamic dGPU passthrough for qemu on Ubuntu. Everything in the hooks folder belongs in `/etc/libvirt/hooks`.
The two shutdown scripts are for shutdown protection in Ubuntu 24.04 LTS, and `org.gnome.Shell@wayland.service` contains a block to be added to the service on Ubuntu to prevent `gnome-shell` from creating a process on the NVIDIA dGPU on startup.

### Specs of computer used for this guide
Lenovo Legion Pro 7 16IAX10H laptop with Intel Ultra 9 275HX, NVIDIA 5080 Max-Q, 32GB RAM, and 1TB NVMe SSD

### OS
Ubuntu 24.04 LTS, Windows 11 25H2 (virtual machine)

### Software versions used
NVIDIA 580-open, virtio-win-0.1.285.iso, Looking Glass B7 

### Goal
Functional Ubuntu with its desktop environment running on the Intel integrated GPU (iGPU), with the ability to run compute tasks on the Nvidia dedicated GPU (dGPU), along with a Windows virtual machine (VM) accessed via Looking Glass that can bind to the dGPU on start-up for graphics intensive tasks, such as Altium, and release it back to Ubuntu when shut down. 

### Notes
Ubuntu will not be able to use the dGPU at all when the VM is on. This is expected behavior due to the implementation of the gpu-passthrough. If the dGPU is needed for tasks in Ubuntu, make sure to first shut down the VM. This guide is written for beginners in mind and therefore designed to minimize terminal navigation/use, with nano used to edit files.

### Known issues
Ubuntu crashes when external display is connected. If the Windows VM is on, it crashes when an external display is connected (but Ubuntu will be fine). Requires two restarts if the display driver ever crashes or is unavailable when Ubuntu is shut down or restarted (Can happen if VM crashes without handing the display driver back to Ubuntu). Brief black screens after Ubuntu starts and after logging in. Sometimes if the computer is left off for too long or goes to sleep too many times (unconfirmed), the VM will crash and cause the GPU drivers to be unavailable, required a multi-restart process.

## 1. BIOS setup and Ubuntu installation

Format a >8GB flash drive as FAT32 for maximum compatibility with UEFI/BIOS, download the Ubuntu 24.04 LTS .iso, and copy it onto the flash drive. https://releases.ubuntu.com/noble/. Shut down the computer and leave the flash drive plugged in. 

### BIOS settings

Power on the computer and enter the BIOS by repeatedly pressing F2 (may be Del on some systems).

Navigate to the Boot section in your BIOS and change the boot order so that the flash drive is first. While in the BIOS, ensure that the virtualization and Intel VT-d is enabled and the graphics are on dynamic mode. Save the BIOS settings and exit.

The computer should now boot from the flash drive and load the Ubuntu .iso. 

### Install Ubuntu

Follow the instructions to install Ubuntu 24.04 LTS on the primary drive. Do not select the option to install additional drivers or codecs. These can be installed as needed afterwards.

Select the option to wipe the entire drive during installation (disregard if the drive has custom partitions set up). Proceed with the installation and follow the instructions on screen.

## 2. Install drivers and clear processes from dGPU 

Open terminal and run `sudo apt update && sudo apt upgrade` to update all current packages. 

### Install NVIDIA drivers 

Check for available NVIDIA drivers. In this example, the NVIDIA 580 open-source drivers have been used, but the latest drivers should work as well.

```bash
sudo ubuntu-drivers list
sudo ubuntu-drivers install nvidia:580-open
```

Reboot, then run `nvidia-smi` in the terminal to confirm that the drivers are loaded. 

### Make sure that prime-select mode is set to on-demand. 

Run:
```bash
sudo apt install nvidia-prime 
prime-select query 
```

If this doesn’t output `on-demand`, run `prime-select on-demand` and reboot. 

Upon rebooting, do not log in yet. Select the user on the start screen and on the following password screen, click on the gear in the bottom right and make sure that "Ubuntu on Wayland" is selected. If the only options are "Ubuntu on Xorg" or "Ubuntu", make sure that "Ubuntu" is selected. This guide is only designed to work with the default: Ubuntu GNOME on Wayland.

Run:

```bash
sudo dmesg | grep -E "VT-d|AMD-Vi"
```

This should show `VT-d active`. If it doesn’t, make sure that VT-d is turned on in the BIOS and dynamic graphics is enabled, if that setting is available. 

Next, run:

```bash
nvidia-smi
```

This should show `No running processes found`. Also check that:

```bash
sudo fuser -v /dev/nvidia* 2>/dev/null
```
outputs nothing. 

If any of these show that Xorg has a process open, log out and make sure that Wayland is selected on the login screen. 

### In the likely case that these show that `gnome-shell` has an active process on the dGPU, add a GNOME shell override: 

Run:
```bash
systemctl --user edit org.gnome.Shell@wayland.service
```

Add the following lines, found in `org.gnome.Shell@wayland.service` in this repo: 

```bash
[Service] 
ExecStartPre=/usr/bin/mkdir -p /tmp/egl_vendor.d 
ExecStartPre=/usr/bin/rm -f /tmp/egl_vendor.d/10_nvidia.json 
ExecStartPre=/usr/bin/ln -fs /usr/share/glvnd/egl_vendor.d/50_mesa.json /tmp/egl_vendor.d/50_mesa.json 
Environment=__EGL_VENDOR_LIBRARY_DIRS=/tmp/egl_vendor.d 
ExecStartPost=/usr/bin/ln -fs /usr/share/glvnd/egl_vendor.d/10_nvidia.json /tmp/egl_vendor.d/10_nvidia.json 
```

Run:

```bash
systemctl --user daemon-reload
``` 

Reboot. Double check that Wayland is selected on the login screen. 

Double check: 

```bash
nvidia-smi
sudo fuser -v /dev/nvidia* 2>/dev/null
```

These should now show that no processes are running on the dGPU. 

## 3. Setup for the virtual machine 

### Install virt-manager to manage the virtual machine 

Install dependencies:

```bash
sudo apt install libvirt-daemon-system libvirt-clients qemu-kvm qemu-utils virt-manager ovmf 
```

Edit GRUB by opening `/etc/default/grub` as root 

```bash
sudo nano /etc/default/grub
```

Find this line: 

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

and change it to  

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt"
```

Update GRUB: 
```bash
sudo update-grub
``` 

Open `/etc/initramfs-tools/modules` as root and add the following lines to the end:
```bash
vfio 
vfio_iommu_type1 
vfio_pci 
vhost-net 
```

Make sure these are loaded on boot with: 

```bash
sudo update-initramfs –u
```

Reboot and run:
```bash
sudo dmesg | grep -e DMAR -e IOMMU
```

This should output a line that says 
```bash
DMAR: Intel(R) Virtualization Technology for Directed I/O
```

### Set up the virtual machine 

Download the latest Windows 11 .iso: https://www.microsoft.com/en-us/software-download/windows11 

Download the latest .iso version of virtio-win-guest-tools: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/. 

Start virt-manager (Virtual Machine Manager). Select File > New Virtual Machine. In the pop-up window, select "Local install media" and select Forward.

On the next page (Step 2), browse for the Windows 11 .iso downloaded earlier and select Forward. 

On the next page (Step 3), choose the appropriate memory and CPUs for the virtual machine. On a 32GB machine with an Ultra i9 275HX (8 performance cores, 16 efficiency cores), a good place to start would be 16GB of RAM and 8 CPU cores. 

On the next page (Step 4), make sure “Enable storage for this virtual machine” and “Create a disk image for the virtual machine” is selected. On a 1TB drive, 256GB, 384GB, or 512GB are reasonable places to start. 

On the next page, (Step 5), name the virtual machine something appropriate. The default name of “win11” will be used for the remainder of this guide. If any changes are made to the name, ensure that “win11” in any further steps is replaced with the selected name. Select “Customize configuration before install”. 

Open the Details page for the new VM. In the CPU tab, make sure that “Copy host CPU configuration (host-passthrough)” is checked. Open the Topology dropdown and check “Manually set CPU topology”. Set Sockets to 1, Cores to the desired number of cores, and Threads to 1. 

In the Disk drive tab, make sure that the Disk bus is set to virtio. 

In the NIC :xx:xx:xx:xx tab, make sure that Device model is set to virtio. 

In the TPM tab, make sure that Type is Emulated, Model is CRB, and Version is 2.0. 

Select the Tablet device and remove it.

Select Add Hardware at the bottom left, select the Storage tab, and select “Select or create custom storage”. Select Manage... and choose the virtio-win-x.x.xxx.iso downloaded earlier. 

## 4. Install Windows on the virtual machine 

After configuring the VM, go to Virtual Machine > Run. Press enter when prompted in the BIOS to boot from the Windows 11 .iso. 

Go through the Windows installation steps. At the point where Windows asks for a drive to be installed on, the VirtIO storage drivers will need to be installed first for the drive to be recognized.

Select Load driver, browse to the VirtIO CD-ROM, and navigate to `viostor/w11/amd64`. Select this folder and the drive should appear with the size selected during virtual machine setup. 

Proceed with the Windows installation as usual. When the computer reboots into the installer with the Windows 11 UI and prompts for a country, open the command line with Shift + F10 and run:

```ps1
oobe/bypassnro
```

This will reboot the installer and make it so that Windows can be installed without internet or a Microsoft account. These can be added later as needed. 

## 5. Setup GPU-passthrough 

Download the files in this repo by clicking the “<> Code” dropdown and then Download ZIP.

Open the Ubuntu file manager and select the address bar. Type `admin://` to open as root and navigate to `/etc/libvirt/hooks`. Copy everything in the `hooks` folder from the downloaded GitHub repo to `/etc/libvirt/hooks`. 

Find the GPU IOMMU group by running 
```bash
find /sys/kernel/iommu_groups/ -type l -exec basename {} \; | sort | xargs -I % lspci -nns %
```

This will print a table of PCI address and devices. Find the addresses that correspond to VGA compatible controller and Audio device for the dGPU. Ideally the VGA compatible controller and Audio device will both be in the same group with no other devices in the group. In this guide, the PCI addresses used are 02:00.0 and 02:00.1 for the VGA compatible controller and the Audio device, respectively. If there are other devices in the group, check other resources for how to proceed before continuing.

Open the virtual machine details in virt-manager and add the GPU VGA compatible controller and Audio device by referencing the PCI addresses found previously.

Before starting the virtual machine, run: 

```bash
lspci -k | grep -A 3 -i "NVIDIA"
```
and verify that for the VGA compatible controller, the `Kernel driver in use:` lists `nvidia` and for the Audio device, the `Kernel driver in use:` lists `snd_hda_intel`. The exact drivers in use may vary based on the machine, but they should not say `vfio-pci`.

Start the virtual machine.  

In Ubuntu, run: 

```bash
lspci -k | grep -A 3 -i "NVIDIA"
```

and verify that for the VGA compatible controller, the `Kernel driver in use:` lists `vfio-pci` and for the `Audio device`, the `Kernel driver in use:` lists `vfio-pci`. 

### After booting into Windows, install network drivers

Open Device Manager, right click the Network adapter, select Update, then in the pop-up window, select “Browse my computer for driver software.” On the next screen, browse to the virtio-win CD-ROM. In the CD-ROM, navigate to `NetKVM/w11/amd64` and install the driver from there. Internet access should now be restored. 

While in Device Manager, also install the keyboard drivers: Right click the keyboard, select Update, then “Browse my computer for driver software.” On the next screen, browse to the virtio-win CD-ROM. In the CD-ROM, navigate to `vioinput/w11/amd64` and install the driver from there. Without doing this, the keyboard should still work but may not be able to pass through keys being held. 

### Install NVIDIA Drivers 

Download the latest NVIDIA drivers by entering the information for the dGPU on the NVIDIA driver website and select the Studio or Game-Ready drivers depending on the intended use-case for the Windows VM. https://www.nvidia.com/en-us/drivers/ 

Proceed with the driver installation. Probably do not select the USB-C driver unless it is needed (untested for this guide). 

After installation, restart the Windows VM, open Task Manager, and check the Resources tab. The NVIDIA dGPU should show up with the expected specifications. Also check Device Manager and make sure that the NVIDIA GPU shows up there under Display as well. The Windows VM should now be able to use the GPU. 

## 6. Set up Looking Glass for near-native response time and performance. 

### Build the Looking Glass client on Ubuntu 

Install dependencies: 

```bash
sudo apt install binutils-dev cmake fonts-dejavu-core libfontconfig-dev gcc g++ pkg-config libegl-dev libgl-dev libgles-dev libspice-protocol-dev nettle-dev libx11-dev libxcursor-dev libxi-dev libxinerama-dev libxpresent-dev libxss-dev libxkbcommon-dev libwayland-dev wayland-protocols libpipewire-0.3-dev libpulse-dev libsamplerate0-dev
```

Download the source files: https://looking-glass.io/downloads. Go to the latest stable version in the table and select Source. Decompress the downloaded file and navigate into it. Navigate to client, then create a folder named build. 

In the build folder (`looking-glass-b7/client/build`) open a terminal and run:

```bash
cmake ../ 
make
```

Install Looking Glass with:

```bash
sudo make install
```

### Set up libvirt and QEMU (IVSHMEM with KVMFR) 

Install the Linux kernel headers:

```bash
sudo apt install linux-headers-$(uname -r) dkms
```

Browse to `looking-glass-b7/module` and run:

```bash
dkms install "."
``` 

Create `/etc/modprobe.d/kvmfr.conf` and add the following line, replacing `128` with the desired amount of memory calculated earlier
```bash
options kvmfr static_size_mb=128
```

Run: 

```bash
modprobe kvmfr
```

Then, run:

```bash
dmesg | grep “kvmfr”
```

The following line should show up. This may or may not work, but if it does, that's a good sign.

```bash
kvmfr: creating 1 static devices
```

Create `/etc/modules-load.d/kvmfr.conf` and add the following lines so that KVMFR loads at boot. 

```bash
# KVMFR Looking Glass module 
kvmfr 
```

## 7. Permissions 

Run:

```bash
ls -l /dev/kvmfr0
```

and something similar to the following output should appear (the file may or may not be owned by root):

```bash
crw------- 1 root root 242, ... /dev/kvmfr0
```

It is important that the line starts with `crw-------` before proceeding. This indicates that `kvmfr0` is a character file.

Run:

```bash
sudo chown user:kvm /dev/kvmfr0
```

replacing `user` with the Ubuntu user's username to set the permissions on the file correctly.

Create the file `/etc/udev/rules.d/99-kvmfr.rules` and add the following line, replacing `user` the Ubuntu user’s username: 

```bash
SUBSYSTEM=="kvmfr", OWNER="user", GROUP="kvm", MODE="0660"
```

Create `/etc/apparmor.d/local/abstractions/libvirt-qemu` and add:

```bash
# Looking Glass 
/dev/kvmfr0 rw, 
```

Edit `/etc/libvirt/qemu.conf`. Find the `cgroup_device_acl`, uncomment it, and add `"/dev/kvmfr0"` to the end of the list. Make sure to add a comma to the previous entry in the list. 

Restart livirtd: 

```bash
sudo systemctl restart libvirtd.service
```

## 8. Virt-manager XML edits for libvirt 

Edit the `<domain>` tag near the top of the XML file to be:

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

At the bottom of the XML file, after the closing `</devices>` block and before the closing `</domain>` block, add the following XML block. The number after size should be the calculated memory size from earlier converted from MB to B. (size_in_bytes = size_in_MB x 1024 x 1024)

```xml
<qemu:commandline> 
 <qemu:arg value="-device"/> 
 <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/> 
 <qemu:arg value="-object"/> 
 <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':33554432,'share':true}"/> 
</qemu:commandline> 
```

For audio passthrough, edit the sound and audio blocks to be: 

```xml
<sound model='ich9'> 
 <audio id='1'/> 
</sound> 
<audio id='1' type='spice'/> 
```

For clipboard synchronization between the VM and Ubuntu, install SPICE guest tools onto the Windows VM (https://www.spice-space.org/download.html#windows-binaries) and edit the spicevmc channel block in the VM’s XML to: 

```xml
<channel type="spicevmc"> 
 <target type="virtio" name="com.redhat.spice.0"/> 
 <address type="virtio-serial" controller="0" bus="0" port="1"/> 
</channel> 
```

Find the <memballoon> block in the VM’s XML and replace the entire block with: 

```xml
<memballoon model="none"/> 
```

Reboot. 

### Install the Looking Glass host on the Windows VM  

Start the Windows VM and download the latest stable Looking Glass Windows host binary: https://looking-glass.io/downloads. Install it as administrator and follow the prompts. 

### Add clean shutdown scripts to prevent errors when restarting or shutting down 

Put `shutdown-win11-looking-glass` from this repo in `/usr/local/sbin`. Replace `win11` everywhere in the file and file name with the VM name chosen earlier if it is different.

Make this file executable with 

```bash
sudo chmod +x /usr/local/sbin/shutdown-win11-looking-glass
```

Put `win11-clean-shutdown.service` from this repo in `/etc/systemd/system` 

Enable it by running: 

```bash
sudo systemctl daemon-reload 
sudo systemctl enable –now win11-clean-shutdown.service
```

Check that it is active by running 

```bash
systemctl status win11-clean-shutdown.service
```

Reboot and confirm that the system doesn't hang. If successful, the virtual machine should be working with Looking Glass and GPU Passthrough! Make sure to always start the virtual machine before starting Looking Glass. Always shut down the virtual machine before shutting down Ubuntu.

## Optional tweaks 

Edit the Looking Glass capture key from the default ScrollLock to Insert by opening `/etc/looking-glass-client.ini` and adding the following:

```ini
[input] 
escapeKey=KEY_INSERT
```

`KEY_INSERT` can be replaced with any desired key. In this case, the Lenovo Legion 7 Pro does not have a dedicated ScrollLock key, so Insert was used instead. 

Hold the escapeKey (Insert in this case) set in the config file while Looking Glass is open to see a list of useful shortcuts, including how to make Looking Glass fullscreen. 

Remap the Copilot key (F23+LeftShift+LeftMeta) to RightControl by installing keyd: 

```bash
sudo apt install keyd
```

Then edit `/etc/keyd/default.conf` by adding this:

```ini
[main] 
f23+leftshift+leftmeta = rightcontrol
```

A personal recommendation would be to change the CapsLock key to another Backspace by adding `capslock = backspace` to the `[main]` block. 

## Resources: 

https://github.com/slackdaystudio/qemu-gpu-passthrough 

https://looking-glass.io/docs/B7

https://releases.ubuntu.com/noble/

https://www.microsoft.com/en-us/software-download/windows11

https://www.nvidia.com/en-us/drivers/

https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/

https://looking-glass.io/downloads

https://www.spice-space.org/download.html
