# Arch Linux Install Cheatsheet

_Leonid Titov, 2024-08-08_

- https://www.youtube.com/watch?v=FFXRFTrZ2Lk
- https://www.youtube.com/watch?v=8T0vvf1xm58
- https://www.youtube.com/watch?v=sG-dGmvZHBo

# Part 1

## Create a bootable USB stick

## Boot & verify the boot mode

	# ls /sys/firmware/efi/efivars

If the command shows the directory without error, then the system is booted in UEFI mode.
If the directory does not exist, the system may be booted in BIOS (or CSM) mode.
If the system did not boot in the mode you desired, refer to your motherboard's manual.

	# efivar --list


## Connect to the Internet

	# ip link
	# ip a


## Check the internet is connected

	# ping archlinux.org
	# ping ya.ru


## Update the system clock

	# timedatectl set-ntp true


## Partition the disks and make filesystems

	# lsblk
	# fdisk -l

Suppose you have a single /dev/sda disk.

-- If automated, better dd random data into it first, to erase previous partitions.

You have to create at least two partitions, EFI system partition (260+ MiB), and Linux root partition.

	# fdisk /dev/sda
	m
	g
	w

then (approximate, be careful!):

	# fdisk /dev/sda
	n
	+300M           -- 100M min
	t
	1
	w

then (approximate, be careful!):

	# fdisk /dev/sda
	n
	t
	23
	w

The result must be:

	/dev/sda1 EFI system partition 300M
	/dev/sda2 Linux x86-64 root the remainder

Now, make filesystems:

	# mkfs.fat -F32 /dev/sda1
	# mkfs.ext4 /dev/sda2

The UEFI specification mandates support for the FAT12, FAT16, and FAT32 file systems[4].
To prevent potential issues with other operating systems and also since the UEFI specification only mandates
supporting FAT16 and FAT12 on removable media[5], it is recommended to use FAT32.

If you get the message WARNING: Not enough clusters for a 32 bit FAT!, reduce cluster size with
mkfs.fat -s2 -F32 ... or -s1; otherwise the partition may be unreadable by UEFI. See mkfs.fat(8)
for supported cluster sizes.

### Non-UEFI

- The same thing, except don't create the ESP/EFI/whatever partition.
- Also, don't bother creating GPT, stick with MBR.
- Don't forget to set bootable flag to main partition.


## Mount the file systems

First time, do this, in this order:

	# mount /dev/sda2 /mnt
	# cd /mnt
	# mkdir boot
	# mount /dev/sda1 boot

Next time, do this, in this order:

	# mount /dev/sda2 /mnt
	# mount /dev/sda1 /mnt/boot


## Swap partition (optional)

If you created a swap partition, e.g. /dev/sda3, then you can install it too into the system:

	# mkswap /dev/swap_partition
	
	# swapon /dev/swap_partition


# Part 2.

## Install essential packages

	# pacstrap /mnt base linux linux-firmware

	# pacstrap /mnt openssh rsync sudo iproute2 dhclient man-db man-pages texinfo nano socat netcat curl wget iptables nftables

Optional:

	# pacstrap /mnt ddrescue diffutils ethtool hdparm nmap
 	# pacstrap /mnt htop wireguard-tools nodejs

Also add grub, if you're installing non-UEFI system.

dhclient is ISC dhclient, specifically.

## Fstab

	# genfstab -U /mnt >> /mnt/etc/fstab

(!) This command must be called exactly __once__, or you'll need to edit the file manually.

Check the resulting /mnt/etc/fstab file, and edit it in case of errors.

__The "relatime" is not a typo!__<br>
relatime updates the access time only if the previous access time was earlier than the current
modify or change time. In addition, since Linux 2.6.30, the access time is always updated if the
previous access time was more than 24 hours old. This option is used when the defaults option,
atime option (which means to use the kernel default, which is relatime; see mount(8) and
wikipedia:Stat (system call)#Criticism of atime) or no options at all are specified.

Note:<br>
If you have several data volumes, and want them be included in fstab - please mount them
beforehand somewhere in the /mnt too.

E.g.:

	sda1 /mnt
	sda2 /mnt/data_volume_A
	sdb1 /mnt/data_volume_B

## Change root into the new system

	# arch-chroot /mnt

## Time zone

	# ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime

	# hwclock --systohc



## Hostname

	# echo <hostname> > /etc/hostname

Add matching entries to hosts(5):

	/etc/hosts
		127.0.0.1 localhost
		::1 localhost
		127.0.1.1 myhostname.localdomain myhostname

If the system has a permanent IP address, it should be used instead of 127.0.1.1.


## Root password & add users

	# passwd
	
	# useradd -m op1
	# passwd op1


## Boot loader

	# bootctl --path=/boot install

Configure it:

	# nano /boot/loader/loader.conf

		timeout 3
		default arch.conf

	# nano /boot/loader/entries/arch.conf

		title Arch Linux
		linux /vmlinuz-linux
		initrd /initramfs-linux.img
		options root=/dev/sda2 rw

### Non-UEFI

	# pacman -S grub
	
	# grub-install --target=i386-pc /dev/sdX

where /dev/sdX is the disk (not a partition) where GRUB is to be installed.

Now you must generate the main configuration file.

	# grub-mkconfig -o /boot/grub/grub.cfg


## Exit & Reboot

	# exit

	# systemctl reboot

Exit the chroot environment by typing exit or pressing Ctrl+d.

Optionally manually unmount all the partitions with umount -R /mnt: this allows noticing any "busy" partitions,
and finding the cause with fuser(1).

Finally, restart the machine by typing reboot: any partitions still mounted will be automatically unmounted
by systemd. Remember to remove the installation medium and then login into the new system with the root account.



# Part 3.

## Network config

	# systemctl enable systemd-networkd
	# systemctl start systemd-networkd
	# systemctl status systemd-networkd

See the list of network interfaces:

	# ip a

Enable DHCP on enp1s0:

	# nano /etc/systemd/network/LAN1.network

		[Match]
		Name=enp1s0
		
		[Network]
		DHCP=yes


Enable static IP on enp2s0:

	# nano /etc/systemd/network/LAN2.network

		[Match]
		Name=enp2s0
		
		[Network]
		Address=10.50.0.67/8
		#Gateway=10.1.10.1
		#DNS=10.1.10.1
		#DNS=8.8.8.8

Restart networkd:

	# systemctl restart systemd-networkd

Install and start DHCP client on enp1s0:

	# systemctl enable dhclient@enp1s0.servive
	# systemctl start dhclient@enp1s0.servive
	# systemctl status dhclient@enp1s0.servive

Check:

	# ip a

While waiting for an IP to be assigned you can run something like:

	# ping <some-host>
	# watch -n 1 ping -c 1 archlinux.org


## Install xorg, firefox, x11vnc, nodm

	# pacman -S xorg firefox x11vnc nodm xterm

To test that X works, re-log as a user, then try:

	$ startx

The xterm must open. Note, the keyboard works only when mouse is over the window.

TIP:

- Ctrl+Alt+F1 to terminal
- Alt-F7 back to GUI


## Configure VNC server

https://wiki.archlinux.org/index.php/x11vnc

## Set up nodm & firefox

https://wiki.archlinux.org/index.php/Nodm

After installing nodm, modify the /etc/nodm.conf file. Set the NODM_USER variable to the user which
should be automatically logged in, and change the NODM_XSESSION variable to point to the script that
starts your session. The NODM_XSESSION script must be executable!

Enable nodm.service so nodm will be started on boot:

	# systemctl enable nodm

Log in as a user, and:

	$ nano .xinitrc

		#! /bin/bash
		firefox

	$ chmod +x .xinitrc

Login session:<br>
For proper session handling, create pam.d file with the following content:

	# nano /etc/pam.d/nodm
	
		#%PAM-1.0
		
		auth      include   system-local-login
		account   include   system-local-login
		password  include   system-local-login
		session   include   system-local-login

Edit the nodm.service, and add these lines:

	Restart=always
	RestartSec=5

(Look at /etc/systemd/system or /lib/systemd/system)

then

	# systemctl daemon-reload


## Set up SSH

https://wiki.archlinux.org/index.php/OpenSSH

	# pacman -S openssh
	
	# systemctl enable sshd
	# systemctl start sshd
	# systemctl status sshd

Config is in the:

	/etc/ssh/sshd_config

## To install the KDE

	https://wiki.archlinux.org/title/KDE


## .
