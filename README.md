**Notice** This repository is cloned from Xogium. For the original instructions
see [here] (https://wiki.seeedstudio.com/ODYSSEY-STM32MP135D/)
I have not tested the Grove commands. Please refer to the original manual.
I am using Debian and the instructions are therefore a little bit more detailed
for Debian users.

# External buildroot tree for STM32MP135D ODYSSEY #
This external tree is to be used in conjunction with upstream buildroot, 
version 2023.02 or later. Previous versions might work, but are not supported, 
nor recommended.

## How to build ##
### Preparations ###


First, you must ensure all the buildroot dependencies are met. Please 
refer to [The buildroot user manual, chapter 2: System 
requirements](https://buildroot.org/downloads/manual/manual.html#requirement). 
[snagboot](https://github.com/bootlin/snagboot) is also required if you 
wish to do eMMC and or NFS boot (see below). 

```bash
sudo apt install sed make binutils build-essential diffutils gcc g++ bash patch gzip bzip2 perl tar cpio unzip rsync bc findutils wget git gtkterm
```

Then, setup the required source code, as follows:
```
wget https://buildroot.org/downloads/buildroot-2026.08.tar.gz
tar -xf https://buildroot.org/downloads/buildroot-2026.08.tar.gz
mv buildroot-2026.08 buildroot
git clone https://github.com/GauiStori/buildroot-stm32mp135d-odyssey.git
cd buildroot-stm32mp135d-odyssey
git switch odyssey-2023-10
```

### Building ###
Once the source code has been set up correctly, you can proceed with the 
build:
```bash
cd buildroot
make BR2_EXTERNAL=/absolute/path/to/buildroot-stm32mp135d-odyssey stm32mp135d_odyssey_emmc_defconfig
make
```
for emmc image and
```bash
make BR2_EXTERNAL=/absolute/path/to/buildroot-stm32mp135d-odyssey stm32mp135d_odyssey_sdcard_defconfig
make
```
for sdcard image.

If everything goes well, you should now have a successfully built 
system in the output/images directory of your buildroot tree.
```bash
ls -1 output/images
combined-tf-a-and-fip.img
emmc.img
fip.bin
rootfs.ext2
rootfs.ext4
rootfs.tar
sdcard.img
stm32mp135d-odyssey-grove.dtb
tee.bin
tee-header_v2.bin
tee-pageable_v2.bin
tee-pager_v2.bin
tf-a-stm32mp135d-odyssey.stm32
u-boot.dtb
u-boot-nodtb.bin
zImage
```

### How to use the system ###
#### eMMC Flash ####

### Snagtool installation

I really recommend not to run any operations on SBCs as root or sudo. Add yourself to the plugdev group and add the
following files on your computer:
```bash
sudo nano /etc/udev/rules.d/70-stm32-dfu.rules
```
containing
```bash
sudo nano SUBSYSTEM=="usb", ATTR{idVendor}=="0483", ATTR{idProduct}=="df11", MODE="0660", GROUP="plugdev", TAG+="uaccess"
```

and
```bash
sudo nano /etc/udev/rules.d/71-stm32-dfu.rules
```
containing
```
SUBSYSTEM=="block", ENV{ID_BUS}=="usb", ENV{ID_VENDOR_ID}=="0483", ENV{ID_MODEL_ID}=="5720", MODE="0660", GROUP="plugdev"
SUBSYSTEM=="block", ENV{DEVTYPE}=="disk", ENV{ID_BUS}=="usb", ENV{ID_VENDOR_ID}=="0483", ENV{ID_MODEL_ID}=="5720", SYMLINK+="odyssey-emmc"
```

Reload rules
```bash
sudo udevadm control --reload-rules
```

Tools needed on Debian 
```bash
sudo apt install dfu-util python3-venv
```

Create virtual environment. All snagtools must be run inside venv.
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install snagboot
```

### Flashing the eMMC ###

Remove the middle boot jumper to be sure STM32 ROM DFU mode is active.
Connect a usb to serial console cable, and apply power to your board over
a usb-c cable to your PC. 

Then, execute the following command from the snagboot package and be
prepared to interrupt the boot sequence when reaching u-boot, by
pressing any key in the serial console window:
```
cd buildroot
snagrecover -s stm32mp13 -f ../buildroot-stm32mp135d-odyssey/board/stm32mp135d-odyssey/utilities/stm32mp1-stm32mp135d-odyssey.yaml
```
U-Boot now waits 5 seconds before autobooting, so the prompt is easier to
catch in gtkterm.

Once at the u-boot prompt, type the following to enable the eMMC boot
partition: ```mmc partconf 1 1 1 1```. This enables the first eMMC boot
partition and ensures it is possible to boot from it, by modifying ext
csd register 179.

Then type ```dfu 0``` in U-Boot to expose the U-Boot DFU alternate
settings to your host machine, including the eMMC boot regions. This is
separate from STM32 ROM DFU. If you run ```dfu-util -l``` too early, you
will only see the ROM DFU entries, for example:
```bash
Found DFU: [0483:df11] ver=0200, devnum=5, cfg=1, intf=0, path="1-5", alt=1, name="@virtual /0xF1/1*512Ba", serial="0005002E3232510937393835"
Found DFU: [0483:df11] ver=0200, devnum=5, cfg=1, intf=0, path="1-5", alt=0, name="@FSBL /0x01/1*128Ke", serial="0005002E3232510937393835"
```

After ```dfu 0```, the eMMC boot regions should appear as alt settings.
They can be listed using the dfu-util command:
```bash
dfu-util -l
Found DFU: [0483:df11] ver=0200, devnum=7, cfg=1, intf=0, path="3-3", alt=4, name="mmc1_boot2", serial="0021001A3232510937393835"
Found DFU: [0483:df11] ver=0200, devnum=7, cfg=1, intf=0, path="3-3", alt=3, name="mmc1_boot1", serial="0021001A3232510937393835"
...
```

Then, use the snagflash tool to write the combined bootloader image into
both boot regions. These commands require the U-Boot DFU alt settings
above, especially alt 3 and 4:
```bash
cd buildroot
snagflash -P dfu -p 0483:df11 --dfu-keep -D 3:output/images/combined-tf-a-and-fip.img
snagflash -P dfu -p 0483:df11 -D 4:output/images/combined-tf-a-and-fip.img
```

Once this is done, reset the board and confirm that the eMMC boot now 
works as expected by putting the middle boot jumper back onto the board. 
It will error out at booting from mmc1 partition 0 given the user area 
is blank, but this is normal at this stage.

When you're back at the u-boot prompt again, type ```ums 0 1``` to 
expose the user area of the eMMC as a usb mass storage device to your 
host machine. Use lsblk to determine which device node it was assigned, 
and replace sdX in the following command with the appropriate device 
node. Double check to ensure you will write on the correct device as it 
will be wiped entirely!
```snagflash -P ums -s output/images/emmc.img -b /dev/sdX```

When the writing has completed, press ctrl+c at the u-boot prompt to 
terminate the usb mass storage mode. Then, reset your board again, and 
confirm that it is now booting linux and that you get a login prompt. 
Log in with the root user and the password 'grove'.

#### Micro sd card boot ####

### SD-card holder permissions

It is highly recommended not to flash the sd-card as root or using sudo.
To avoid this do the following.
Connect your USB card holder to your computer and run 
```bash
udevadm info -q property -n /dev/sdc | grep -E 'ID_BUS|ID_VENDOR_ID|ID_MODEL_ID|ID_SERIAL|ID_DRIVE_FLASH_SD'
```
This results in my case with
```
ID_BUS=usb
ID_MODEL_ID=b7ba
ID_SERIAL=SanDisk_SDDR-199_20080000048-0:0
ID_SERIAL_SHORT=20080000048
ID_VENDOR_ID=0781
```
```bash
sudo nano /etc/udev/rules.d/70-sdcard.rules
```
containing
```
# SanDisk SDDR-199 SD-card reader
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_BUS}=="usb", ENV{ID_VENDOR_ID}=="0781", ENV{ID_MODEL_ID}=="b7ba", ENV{ID_SERIAL_SHORT}=="20080000048", GROUP="plugdev", MODE="0660"
```
and then reload the rules
```bash
sudo udevadm control --reload-rules
```
and the SD-card device ```/dev/sdX``` will be writable by group plugdev.

### Flashing

If you wish to burn the system onto a micro sd card, proceed as 
follows, replacing sdX with the appropriate device node:
```dd if=output/images/sdcard.img of=/dev/sdX bs=4M conv=fsync```
Where sdX corresponds to the micro sd card device node.

Remove all jumpers to select sd-card boot.

When the micro sd card has been successfully written to, insert it into 
the micro sd socket of the STM32MP135D ODYSSEY board, and apply power 
via usb-c or PoE. The system will print to the serial console by 
default, so make sure to connect a usb to serial console cable. Log in 
with the user root and the password 'grove'.

#### Nfs boot ####
To boot the system via NFS, please ensure to set up your /etc/exports as 
demonstrated, replacing the subnet / allowed ip addresses as needed, 
doing the same for the exported paths:
```
/srv/nfs 192.168.1.0/24(rw,sync,crossmnt,fsid=0)
/srv/nfs/stm32mp135d 192.168.1.0/24(rw,nohide,insecure,no_subtree_check,async,no_root_squash)
```

Also make sure your nfs server configuration enables UDP mode, like so:
```
/etc/nfs.conf
[nfsd]
...
udp=y
```

Remove the middle boot jumper to be sure DFU mode is active. Connect an 
usb to serial console cable, an ethernet cable to eth1 and apply power 
to your board. Make sure to open the serial console using minicom or another 
similar program, you 
will need it.

Then, execute the following command from the snagboot package and be 
prepared to interrupt the boot sequence when reaching u-boot, by 
pressing any key in the serial console window:
```
cd buildroot
snagrecover -s stm32mp13 -f ../buildroot-stm32mp135d-odyssey/board/stm32mp135d-odyssey/utilities/stm32mp1-stm32mp135d-odyssey.yaml
```
U-Boot now waits 5 seconds before autobooting, so the prompt is easier to
catch in gtkterm.

Once you're at the u-boot prompt, you can boot over nfs by doing the 
following:
```
setenv eth1addr 2c:f7:f1:30:2b:62
setenv ethaddr 2c:f7:f1:30:2b:62
dhcp
nfs ${kernel_addr_r} 192.168.1.92:/srv/nfs/stm32mp135d/boot/zImage
nfs ${fdt_addr_r} 192.168.1.92:/srv/nfs/stm32mp135d/boot/stm32mp135d-odyssey-grove.dtb
setenv bootargs root=/dev/nfs rootfstype=nfs ip=dhcp nfsroot=192.168.1.92:/srv/nfs/stm32mp135d,tcp,v3 rw quiet console=ttySTM0,115200n8 earlycon
bootz ${kernel_addr_r} - ${fdt_addr_r}
```
Where 192.168.1.92 in this example is the machine hosting the nfs 
server. The MAC address set is also an example and not to be used in the 
real world. It is required due to having no MAC addresses defined in the 
OTP, but can be stored semi-permanently in the EEPROM (see below). Log 
in with the user root and no password.

## EEPROM ##
### Layout ###
The current implementation to read MAC addresses from the EEPROM is 
expecting the first one to start at offset 0 and have a length of 6 
bytes. The second MAC must be stored at offset 0x10, and also have a 
length of 6 bytes.

For storing the u-boot environment into the EEPROM if you wish to do so, 
please ensure that the environment begins on a new page boundary. Pages 
are 64 bytes in size. For example, you could set the environment offset 
to 0x40, the size remaining at 0x2000, and redundant offset to 0x2080. 
Here's an example u-boot config fragment:
```
CONFIG_ENV_IS_IN_EEPROM=y
CONFIG_ENV_OFFSET=0x40
CONFIG_ENV_OFFSET_REDUND=0x2080
CONFIG_I2C_EEPROM=y
CONFIG_SYS_I2C_EEPROM_ADDR=0x50
CONFIG_NVMEM=y
```

To apply it, run ```make menuconfig``` in the toplevel buildroot 
directory. Go to the bootloaders menu, scroll down to u-boot and modify 
the additional config fragments path, for example by inputting 
```$(BR2_EXTERNAL_STM32MP135D_ODYSSEY_PATH)/board/stm32mp135d-odyssey/configs/uboot.config```.

Then, please do a rebuild using ```make clean && make```.

Ultimately the layout is free for you to use, except for the MAC 
addresses location and length.

### How to use the EEPROM ###
To make use of the EEPROM on your board, you get access to it as a nvmem 
device, in both u-boot and linux. For example, lets write a MAC address 
in it:
```printf '\x2c\xf7\xf1\30\x2b\x62'|dd of=/sys/bus/nvmem/devices/0-00501/nvmem bs=1```

To store a second MAC address, do like so: ```printf '\x2c\xf7\xf1\30\x2b\x63'|dd of=/sys/bus/nvmem/devices/0-00501/nvmem bs=1 seek=16```

## Networking and SSH ##

Linux will boot and acquire an IP over Ethernet using DHCP. You can then log in
to your device using this command:

```
ssh root@192.168.1.92
```

Replace 192.168.1.92 with your device's IP. You can usually find this in your
router control page.

Once logged in you can change the password using the ```passwd``` command.

Alternatively you can disable SSH access by setting
```BR2_PACKAGE_DROPBEAR=y``` to ```BR2_PACKAGE_DROPBEAR=n``` in your buildroot
```.config``` file, then rebuilding and re-flashing your board.

## Modifying and extending Buildroot ##

Buildroot is a fantastic system for creating embedded Linux systems. If you
plan to use this system I highly recommend reading [the Buildroot user
manual](https://buildroot.org/downloads/manual/manual.html).

That said, here are some helpful tips to get you started:

- 'make menuconfig' will let you add or remove packages
- 'make savedefconfig' updates the configs/ file in this repository
- strace and kernel tracing are enabled by default for this tree
- Put files in rootfs_overlay/etc/init.d/ and mark them as executable to run them on boot
- Put files in rootfs_overlay/usr/bin and mark them as executable to run them from a shell
- Look in the packages directory for examples of packaging C and python packages

If you have questions about this tree or find bugs, please open an issue in the
GitHub repository.

