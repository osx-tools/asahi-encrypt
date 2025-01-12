# NAME

asahi-encrypt - encrypt root partition of Fedora Asahi Remix installation

# SYNOPSIS

List installations available for encryption (example):
    asahi-encrypt -l

Encrypt installation (example):
    asahi-encrypt /dev/nvme0n1p6

# DESCRIPTION

**asahi-encrypt** is script that allows you to encrypt the root partition
of your Fedora Asahi Remix installation inplace. It can run from USB
drive or another installation of Asahi Linux on the same machine.

Features:
- Decent error handling. It unlikely let you go wrong, eg. accidentally
encrypt your running system, or unsuitable partition.
- Resumable. You can interrupt the execution at any point. Just
run again to continue.
- Idempotent. You can run it as many times as you want, even run against
already encrypted system. The changes are mede only if required.
- Provides means to mount and unmount encrypted or unencrypted Asahi
Linux installations to do configuration or maintenance jobs if needed.

Before proceeding:
- Please backup your data
- It is good idea to learn how it works, see OPERATIONS section

The author of the script is not responsible for any harm caused by using
the script.

# USAGE

Usage: asahi-encrypt /dev/nvme0n1p{N} [OPTIONS]
       asahi-encrypt -l [/dev/nvme0n1]

  Where /dev/nvme0n1p{N} is the root partition of Asahi linux installation,
the installation is expected to consist of four partitions:
    /dev/nvme0n1p{N-3} - 2.5G Macos Boot Loader APFS partition (aka Stub-Macos)
    /dev/nvme0n1p{N-2} - 500M EFI partition
    /dev/nvme0n1p{N-1} - 1.0G Boot partition
    /dev/nvme0n1p{N}   -      Root partition, that will be encrypted

  You can easily determine suitable root partitions using 'asahi-encrypt -l'
or 'lsblk -f' command. Root partitions usually have label 'fedora'.

OPTIONS:
    -m  - Just mount target system to /mnt
    -e  - Just encrypt target system, implies '-m'
    -u  - Just unmount target system from /mnt
    -l  - List suitable root partitions and exit
    -h  - Display this help

  If none of '-m', '-e', '-u' options given, all three actions implied, i.e.:
mount, encrypt, unmount

# OPERATIONS

### A brief description of what it does

**1. Shrink root filesystem by 32M**
It is required to accomodate LUKS header when partition will be encrypted.

\# btrfs filesystem resize -32M /mnt

**2. Encrypt root partition inplace**
The root partition is encrypted, partition data will be preserved.

\# cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/nvme0n1p6

**3. Configure target system**
A record for encripted root partition is added to **/etc/crypttab**.
Boot time kernel parameter is added to **/etc/default/grub**.

**4. Rebuild initramfs on target system**
This is necessary to propagate the changes we've made to the components
responsible for booting the target system, so that the root partition
is decrypted at the boot time.

\# chroot /mnt grub2-mkconfig -o /boot/grub2/grub.cfg
\# chroot /mnt dracut -f --kver {KERNEL_VERSION}

`--kver {KERNEL_VERSION}` is needed if the kernels in the running and
target system has different versions. The script detects the last
installed kernel version in the target system and passes it to the
command.

**5. Temporarily disable selinux enforcing**
Without this step system most likely won't boot. Temporarily add
`enforsing=0` to kernel parameters.

\# chroot /mnt grubby --update-kernel ALL --args enforcing=0

Then we schedule rebuilding initramfs on the first boot of the target
system, which reset this enforcing parameter and adjust selinux. The
commands are `grub2-mkconfig -o /boot/grub2/grub.cfg` and `dracut -f`.
The script schedules them via cron.

# RECOMMENDATIONS

I persanally don't like to mess with USB sticks and loaders. I usually
create little Asahi Linux installation of 8Gb (total 12Gb with other
necessary partitions) at the end of the disk. During the installation
process I choose "Minimal system" and name it "Asahi Resque". I then
can use it as a rescue system, for encriptyng, reconfiguring and do the
maintenance jobs of the main installation(s).

### Usage example

First you need to clone the repository ro your USB stick or Rescue OS
(described above):

\# git clone https://github.com/osx-tools/asahi-encrypt.git

Assuming you have booted from the Resque OS or USB stick. Navigate to
`./asahi-encrypt/bin` directory and run:

\# ./asahi-encrypt -l
/dev/nvme0n1p6 btrfs 115.4G fedora

As you can see I have one installation of Asahi Linux on this machine,
which is unencrypted, because it is displayed as `btrfs`. Script will
not show your current running os, because encription of running system
is prohibited. Then we just encrypt this partition with one command:

\# ./asahi-encrypt /dev/nvme0n1p6

You'll be asked for your password if you are running as a regular user,
and a passphrase to encrypt the root partition of the target system.
After the script finishes, you see that now it is encrypted:

\# ./asahi-encrypt -l
/dev/nvme0n1p6 crypto_LUKS 115.4G

Now you can boot to your encrypted system. During the boot you'll be
asked for the passphrase to decrypt root.


### Rescue / Maintenance example

Befor or after or independently of encrypting, you can always mount
and unmount the other installations of Asahi Linux on `/mnt` of your
currently running system. To do so let's find available installations
as in the previous example:

\# ./asahi-encrypt -l
/dev/nvme0n1p6  crypto_LUKS 115.4G
/dev/nvme0n1p10 btrfs        89.3G fedora

You can also use extended view of all your partitions using `lsblk`:

\# lsblk -f

After choosing which target system you want to mount, issue command:

\# ./asahi-encrypt /dev/nvme0n1p10 -m

You will be asked sudo password if running as a regular user and a
passphrase if the target system is encrypted. Now you can chroot to
your target system:

\# chroot /mnt

Do the configuration or maintenance jobs, and after that, run the
following command to unmount the target system:

\# ./asahi-encrypt /dev/nvme0n1p10 -u

Now the target system is unmounted and you can reboot.

# CREDITS

https://davidalger.com/posts/fedora-asahi-remix-on-apple-silicon-with-luks-encryption

# REPORTING BUGS

Bug tracker: https://github.com/osx-tools/asahi-encrypt/issues

# COPYRIGHT

Copyright (c) 2024 albert-a &lt;albert-a@github.com&gt;
License:MIT &lt;https://mit-license.org&gt;

# SEE ALSO

**btrfs-filesystem**(8), **chroot** **cryptsetup-reencrypt**(8), **crypttab**(5), 
**dracut**(8), **grub2-mkconfig**(8).
