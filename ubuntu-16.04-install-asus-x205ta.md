# Instructions to install Ubuntu 16.04 LTS on ASUS EeeBook X205TA

**_ALERT: 16.04.2 installs linux kernel 4.8.0 which breaks a few things like the keyboard. Currently working on a fix_**

## Required items
* Separate system already running Ubuntu 16.04 - this is where you will build the 32bit boot loader and create the USB install flash drive.
* Bootable USB flash drive at least 2GB - ALL DATA WILL BE REMOVED FROM THIS DRIVE!

## Prepare the USB flashdrive to be used as the install media
**THIS WILL DELETE DATA ON /dev/sdb - MAKE SURE YOU KNOW WHAT YOU ARE DOING!**

On a separate system already running Ubuntu 16.04 and powered on, plug in the USB flash drive. Run the following to setup the install media:
```bash
apt -y install p7zip-full

wget http://releases.ubuntu.com/16.04.2/ubuntu-16.04.2-desktop-amd64.iso

# Assuming USB flashdrive assigned to /dev/sdb
# THIS WILL DELETE ALL DATA ON /dev/sdb - make sure you know what you are doing!
sgdisk --zap-all /dev/sdb
sgdisk --new=1:0:0 --typecode=1:ef00 /dev/sdb
mkfs.vfat -F32 /dev/sdb1

# Copy the ISO file to the USB drive:
mount -t vfat /dev/sdb1 /mnt
7z x ubuntu-16.04.2-desktop-amd64.iso -o/mnt/

# Create the 32bit boot loader and place it on USB install media
apt -y install git bison libopts25 libselinux1-dev autogen \
  m4 autoconf help2man libopts25-dev flex libfont-freetype-perl \
  automake autotools-dev libfreetype6-dev texinfo
git clone git://git.savannah.gnu.org/grub.git
cd grub
./autogen.sh
./configure --with-platform=efi --target=i386 --program-prefix=''
make
cd grub-core
../grub-mkimage -d . -o bootia32.efi -O i386-efi -p /boot/grub \
  ntfs hfs appleldr boot cat efi_gop efi_uga elf fat hfsplus iso9660 linux keylayouts \
  memdisk minicmd part_apple ext2 extcmd xfs xnu part_bsd part_gpt search search_fs_file \
  chain btrfs loadbios loadenv lvm minix minix2 reiserfs memrw mmap msdospart scsi loopback \
  normal configfile gzio all_video efi_gop efi_uga gfxterm gettext echo boot chain eval
cp bootia32.efi /mnt/EFI/BOOT/

umount /mnt
```
Remove the USB flash drive.

## Installation on X205TA

### BIOS setup

* Make sure the X205TA is off
* Plug in the USB flash drive
* Start the X205TA and continue to press `F2` to get into BIOS.
* Under `Advanced` tab, `USB Configuration` -> `USB Controller Select` set to `EHCI` otherwise mouse and keyboard will not work
* Under `Security` tab, `Secure Boot menu` -> `Secure Boot Control` set to `Disabled`. Otherwise, you may get a **SECURE BOOT VIOLATION** on boot. This is due to the factory defaut keys not working for Ubuntu's EFI bootloader. Instead of disabling `Secure Boot Control`, you can 1) `Key Management` -> `Delete All Secure Boot Variables` or 2) `Key Management` and manually load all of the variables.  Disabling `Secure Boot Control` was easier. [More information on SecureBoot and Ubuntu](http://web.dodds.net/~vorlon/wiki/blog/SecureBoot_in_Ubuntu_12.10/)
* Under `Save & Exit` tab, `Save Changes` (NOT `Save Chances and Exit`)
* Lastly, while still in `Save & Exit` tab, under `Boot Override`, select the USB flash drive.

### Installation

1. At the grub menu, select "Try Ubuntu without installing" (misleading because you will be installing). This is needed to get a shell to setup the wireless device. Network access is required for installation to remotely obtain the `grub-efi-ia32-bin` package.  Without this package, install will fail.
2. Once the desktop loads, press "ctrl+alt+t" to get a terminal window.
3. Type the following to setup the wireless network device:

   ```bash
   sudo cp /sys/firmware/efi/efivars/nvram-74b00bd9-805a-4d61-b51f-43268123d113 /lib/firmware/brcm/brcmfmac43340-sdio.txt
   sudo modprobe -v -r brcmfmac
   sudo modprobe -v brcmfmac
   ```

4. The wireless network device should now be working. Press alt+F10 (or click on the wireless/network icon on the indicator panel) to configure using your wireless network setup.
5. Double click on the Desktop icon labeled "Install Ubuntu 16.04 LTS" to start the installation process and install as usual.

*Note: Selecting `Encrypt the new Ubuntu installation for security` will require you to enter a password on boot - the keyboard will not work for this requiring you to use an external USB keyboard. You have been warned.*

### Post-installation

* Due to a known system freeze issue, update `/etc/default/grub` by updating the `GRUB_CMDLINE_LINUX_DEFAULT` line to:

  ```
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_idle.max_cstate=1"
  ```

  Then run `update-grub`.
* We will need to copy a file as we did during installation to get wifi working:

  ```bash
  sudo cp /sys/firmware/efi/efivars/nvram-74b00bd9-805a-4d61-b51f-43268123d113 /lib/firmware/brcm/brcmfmac43340-sdio.txt
  sudo modprobe -v -r brcmfmac
  sudo modprobe -v brcmfmac
  ```

  Then configure your wireless settings.

* Stop login freezes caused by generic bluetooth driver btsdio:  

  Many have reported a high frequency of system freezes at the login screen. It was found that if one were to blacklist the btsdio generic bluetooth driver, the login freezes stop.  Add the following to `/etc/modprobe.d/blacklist.conf`:

  ```
  # generic bluetooth SDIO driver
  blacklist btsdio
  ```

  Subsequent login instances should no longer freeze.

## Bluetooth setup

The firmware for the bluetooth hardware needs to be installed and the service started in order to function. The file needed for this is [BCM43341B0.hcd](http://lopaka.github.io/files/instructions/BCM43341B0.hcd)

###Quick note on how firmware file was obtained:
1. From https://software.intel.com/en-us/iot/hardware/edison/downloads click on **Latest Yocto* Poky image** to download a file similar to `iot-devkit-prof-dev-image-edison-20160606-patch.zip`
2. Unzip the file and you will find a file called `edison-image-edison.ext4`
3. Mount this file: `mount edison-image-edison.ext4 /mnt`
4. The file `/mnt/etc/firmware/bcm43341.hcd` is what we are looking for to be renamed `BCM43341B0.hcd`

###Steps to setup bluetooth

```bash
# Download firmware file and install it
wget http://lopaka.github.io/files/instructions/BCM43341B0.hcd -O /lib/firmware/brcm/BCM43341B0.hcd

# Create systemd service file
cat >/etc/systemd/system/btattach.service <<EOL
[Unit]
Description=Btattach

[Service]
Type=simple
ExecStart=/usr/bin/btattach --bredr /dev/ttyS1 -P bcm
ExecStop=/usr/bin/killall btattach

[Install]
WantedBy=multi-user.target
EOL

# Enable service
systemctl enable btattach
```

## *EXPERIMENTAL* Kernel changes for audio support

The section was created thanks to the great work done by those on [UbuntuForums](https://ubuntuforums.org/showthread.php?t=2254322&page=126&p=13592053#post13592053).

The following steps must be done on the x205ta after installation to provide *experimental* audio support:

```bash
# Required lib and packages not installed by default
apt -y install git
apt -y install libssl-dev

# Retrieve the Linux kernel source tree fork - will take some time
git clone https://github.com/plbossart/sound.git -b experimental/codecs
cd sound

# Obtain the kernel config already done - otherwise you will have to run
# 'make localmodconfig', 'make menuconfig', and answer questions.
# Original file from:
#   ftp://x205ta.myftp.org:1337/kernel/.config
wget http://lopaka.github.io/files/instructions/x205ta.config -O .config

# reverse patch the commit that causes the keyboard to malfunction
git diff 3ae02c1^ 3ae02c1 | patch -Rp1

# Add patch that attempts to fix non-functioning FN-keys
# Original file from:
#   https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/fn-brightness-hack.patch
wget http://lopaka.github.io/files/instructions/fn-brightness-hack.patch
patch -p1 < fn-brightness-hack.patch

# Build - will take some time
make -j6

# Install modules
make modules_install

# Install kernel to the boot dir
export KERNELRELEASE=$(<include/config/kernel.release)
cp -va arch/x86/boot/bzImage /boot/vmlinuz-$KERNELRELEASE

# Build initramfs
update-initramfs -c -k $KERNELRELEASE

# Rebuild /boot/grub/grub.cfg
update-grub

# Obtain HiFi.conf and install it at /usr/share/alsa/ucm/chtrt5645/
# Original files from:
#   https://raw.githubusercontent.com/plbossart/UCM/master/chtrt5645/HiFi.conf
#   https://raw.githubusercontent.com/plbossart/UCM/master/chtrt5645/chtrt5645.conf
mkdir -p /usr/share/alsa/ucm/chtrt5645
wget http://lopaka.github.io/files/instructions/HiFi.conf -O /usr/share/alsa/ucm/chtrt5645/HiFi.conf
wget http://lopaka.github.io/files/instructions/chtrt5645.conf -O /usr/share/alsa/ucm/chtrt5645/chtrt5645.conf

# Install audio packages
apt -y install pulseaudio alsa-base alsa-utils pavucontrol

# Reboot and use GUI to set default output - Sound Settings...
shutdown -r now
```
