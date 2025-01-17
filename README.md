# NAME

asahi-encrypt - encrypt root partition of Fedora Asahi Remix installation

# SYNOPSIS

List installations available for encryption (example):

    asahi-encrypt -l

Encrypt the installation (example):

    asahi-encrypt /dev/nvme0n1p6

# DESCRIPTION

**asahi-encrypt** is script that allows you to encrypt the root partition
of your Fedora Asahi Remix installation inplace. It can run from USB
drive or another installation of Asahi Linux on the same machine.

Features:
- Error handling. It is unlikely, that the script will let you go wrong,
eg. accidentally encrypt your running system or unsuitable partition.
- Resumable and idempotent. In case of interruption or power loss at any
point of execution, just run again to continue, it will take care of
recovery/resuming of encryption if needed, all changes are made once, no
matter how many times you run the script.
- Can mount and unmount encrypted or unencrypted Asahi Linux
installations to do configuration or maintenance jobs if needed.

Before proceeding:
- Please backup your data
- It is good idea to learn how it works, see OPERATIONS section
- Check EXAMPLES section for a detailed usage example

The author of the script is not responsible for any harm caused by using
the script.

# USAGE

``` ini
Usage: asahi-encrypt /dev/nvme0n1p{N} [OPTIONS]
       asahi-encrypt -l [/dev/nvme0n1]

  Where /dev/nvme0n1p{N} is the root partition of Asahi linux
installation. The installation is expected to consist of four
partitions:
    /dev/nvme0n1p{N-3} - 2.5G apfs  Macos Boot Loader
    /dev/nvme0n1p{N-2} - 500M vfat  EFI
    /dev/nvme0n1p{N-1} - 1.0G ext4  Boot
    /dev/nvme0n1p{N}   -      btrfs Root (will be encrypted)

  You can easily determine suitable root partitions using
one of the commands: 'asahi-encrypt -l' or 'lsblk -f'.
Unencrypted root partitions of a typical Fedora Linux
installation are usually labeled 'fedora'.

OPTIONS:
    -m  - Just mount target system on /mnt
    -e  - Just encrypt target system, implies '-m'
    -u  - Just unmount target system from /mnt
    -l  - List suitable root partitions and exit
    -h  - Display this help

  If none of the '-m', '-e', '-u' options are given, all
three actions are assumed: mount, encrypt, unmount.

```

# OPERATIONS

A brief description of what it does

## 1. Shrink root filesystem by 32M
It is required to accomodate LUKS header when partition will be encrypted.

``` sh
btrfs filesystem resize -32M /mnt
```

## 2. Encrypt root partition inplace
The root partition is encrypted, partition data will be preserved.

``` sh
cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/nvme0n1p6
```

## 3. Configure target system
A record for encripted root partition is added to **/etc/crypttab**.
Boot time kernel parameter is added to **/etc/default/grub**.

## 4. Rebuild initramfs on target system
This is necessary to propagate the changes we've made to the boot
components of the target system, so that the root partition is decrypted
at the boot time.

``` sh
chroot /mnt grub2-mkconfig -o /boot/grub2/grub.cfg
chroot /mnt dracut -f --kver {KERNEL_VERSION}
```

`--kver {KERNEL_VERSION}` is needed if the kernels in the running and
target system has different versions. The script detects the last
installed kernel version in the target system and passes it to the
command.

## 5. Temporarily disable selinux enforcing
Temporarily add `enforsing=0` to kernel parameters. Without this step
the system most likely won't boot.

``` sh
chroot /mnt grubby --update-kernel ALL --args enforcing=0
```

Then we schedule rebuilding initramfs on the first boot of the target
system, which will reset this parameter. The script schedules the
following commands via cron for this purpose:

``` sh
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut -f
setenforce 1
```

# RECOMMENDATIONS

Rather than preparing an USB stick, it might be much easyer to create a
small 16Gb Asahi Linux installation at the end of the your system disk.
During the installation process, choose "Minimal system" and name it eg.
"Asahi Resque". It can then be used as a rescue system for encryption,
configuration or performing maintenance jobs on the main installation(s).

> [!WARNING]
> Currently, you **can not** install two OSes of the same type because
> of [this](https://pagure.io/fedora-asahi/remix-bugs/issue/9) issue,
> also [here](https://github.com/AsahiLinux/asahi-installer/issues/265).
> Eg. two Gnome or two KDE or two minimal systems. Although installing
> eg. Gnome, KDE and a minimal system side by side is perfectly
> acceptable.

# EXAMPLES

## Usage example

First you need to clone the repository to your USB stick or Rescue OS
(described above):

``` sh
git clone https://github.com/osx-tools/asahi-encrypt.git
```

Assuming you have booted from the Resque OS or the USB stick. Navigate
to `./asahi-encrypt/bin` directory and run:

``` sh
./asahi-encrypt -l
```
>``` txt
>/dev/nvme0n1p6 btrfs 55.8G fedora
>```

As you can see I have one installation of Asahi Linux on my machine
except the running one. The installation is unencrypted, because it is
displayed as `btrfs`. The script will not show your currently running
OS, because encription of running system is not allowed yet. Then we
just encrypt this partition with one command:

``` sh
./asahi-encrypt /dev/nvme0n1p6
```
>``` txt
>>>> Mounting the target system on /mnt
>[sudo] password for albert: 
>>>> Shrinking the target root filesystem to accomodate LUKS header
>Resize device id 1 (/dev/nvme0n1p6) from 55.79GiB to 55.76GiB
>>>> Unmounting the target system from /mnt
>>>> Encrypting the target root partition inplace
>
>WARNING!
>========
>This will overwrite data on LUKS2-temp-57e96d3b-0a1f-4f92-9bfe-446ce96aabe1.new irrevocably.
>
>Are you sure? (Type 'yes' in capital letters): YES
>Enter passphrase for LUKS2-temp-57e96d3b-0a1f-4f92-9bfe-446ce96aabe1.new: 
>Verify passphrase: 
>Finished, time 00m37s,   55 GiB written, speed   1.5 GiB/s
>>>> Opening the encrypted root partition of the target system
>Enter passphrase for /dev/nvme0n1p6: 
>>>> Mounting the target system on /mnt
>>>> Configuring the target system
>>>> Rebuilding initramfs for the target system
>Generating grub configuration file ...
>Found Fedora Linux Asahi Remix 41 (Workstation Edition) on /dev/mapper/fedora-6ad5a9a1
>Adding boot menu entry for UEFI Firmware Settings ...
>done
>dracut[E]: No '/dev/log' or 'logger' included for syslog logging
>>>> Disabling selinux enforcing for the target system until next boot
>>>> Unmounting the target system from /mnt
>>>> Closing the encrypted root partition of the target system
>>>> Done!
>```

You'll be asked for your password if you run it as a regular user and
a passphrase to encrypt the root partition of the target system. After
the script finishes, you will see that the partition is encrypted:

``` sh
./asahi-encrypt -l
```
>``` txt
>/dev/nvme0n1p6 crypto_LUKS 55.8G 
>```

Now you can boot to your encrypted system. During the boot you'll be
asked for the passphrase to decrypt the root partition.

## Mount / unmount example (for maintenance)

Independently of encrypting, you can always mount other Asahi Linux
installations on your machine on `/mnt`. In this example there are two
Asahi linux installations other than currently running one:

``` sh
./asahi-encrypt -l
```
>``` txt
>/dev/nvme0n1p6  crypto_LUKS 55.8G.
>/dev/nvme0n1p14 crypto_LUKS 11.1G.
>```

You can also use extended view of all your partitions using `lsblk -f`:

``` sh
lsblk -f
```
>``` txt
>NAME                FSTYPE      FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
>zram0                                                                                                 [SWAP]
>nvme0n1                                                                                               
>├─nvme0n1p1         apfs                          3abdfbb7-b3ac-4ddf-aa8b-210761d14d2a                
>├─nvme0n1p2         apfs                          d74c6c3b-9740-401b-b210-47628209d907                
>├─nvme0n1p3         apfs                          8f969030-c1ef-4785-979c-fd6718dfe95d                
>├─nvme0n1p4         vfat        FAT32 EFI - ASAHI 2005-46BA                                           
>├─nvme0n1p5         ext4        1.0   BOOT        834529c8-8a29-4101-83a4-dd776922c2c8                
>├─nvme0n1p6         crypto_LUKS 2                 57e96d3b-0a1f-4f92-9bfe-446ce96aabe1                
>├─nvme0n1p7         apfs                          8635584a-0752-4cd0-825d-48aa79b1845d                
>├─nvme0n1p8         vfat        FAT32 EFI - ASAHI 23FA-AD3E                             370.7M    26% /boot/efi
>├─nvme0n1p9         ext4        1.0   BOOT        69fa608d-1dd2-408a-9907-7d0881db5b4a  792.2M    12% /boot
>├─nvme0n1p10        crypto_LUKS 2                 6ad5a9a1-6d8b-4468-b9d9-2696b13bd940                
>│ └─fedora-6ad5a9a1 btrfs             fedora      a4ab00fe-49d8-4f06-b59a-92d35fb5a311  105.8G     6% /home
>│                                                                                                     /
>├─nvme0n1p11        apfs                          a9e974f3-d48a-4344-90ce-07f75529546f                
>├─nvme0n1p12        vfat        FAT32 EFI - ASAHI FEDF-195A                                           
>├─nvme0n1p13        ext4        1.0   BOOT        cc448e29-b2af-4ebb-85fd-e5e0f23604b2                
>├─nvme0n1p14        crypto_LUKS 2                 c9c54c52-8f69-4310-9421-06d49d46a525                
>└─nvme0n1p15        apfs                          342b57f9-0698-44e8-b734-d65f7b66994f                
>nvme0n2                                                                                               
>nvme0n3                                                                               
>```

After choosing which target system you want to mount, issue the command:

``` sh
./asahi-encrypt /dev/nvme0n1p6 -m
```
>``` txt
>[sudo] password for albert: 
>>>> Opening the encrypted root partition of the target system
>Enter passphrase for /dev/nvme0n1p6: 
>>>> Mounting the target system on /mnt
>>>> Done!
>```

You will be asked for your password if you run it as a regular user and
a passphrase if the target system is encrypted. Now you can chroot to
your target system:

``` sh
chroot /mnt
```

Do the configuration or maintenance jobs. When finished, run the
following command to unmount the target system:

``` sh
./asahi-encrypt /dev/nvme0n1p6 -u
```
>``` txt
>>>> Unmounting the target system from /mnt
>>>> Closing the encrypted root partition of the target system
>>>> Done!
>```

Now the target system is unmounted and you can reboot.

# CREDITS

[Fedora Asahi Remix with LUKS Encryption by David Alger](https://davidalger.com/posts/fedora-asahi-remix-on-apple-silicon-with-luks-encryption)

# REPORTING BUGS

https://github.com/osx-tools/asahi-encrypt/issues

# COPYRIGHT

Copyright (c) 2024 abert-a: albert-a@github.com

License: MIT https://mit-license.org

# SEE ALSO

**btrfs-filesystem**(8), **chroot**(1), **cryptsetup-reencrypt**(8), **crypttab**(5),
**dracut**(8), **grub2-mkconfig**(8).
