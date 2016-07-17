# live-usb-maker
Create an antiX/MX LiveUSB
```
Usage: live-usb-maker [options] <iso-file> <usb-device> <commands>

Create a live-usb on <usb-device> from <iso-file>.  This will destroy
any existing information on <usb-device>.

Use ext4 as the filesystem for the live-usb and add a small fat32 file
system for booting via UEFI.

At least one command must be given.  If "all" is not given then only the
commands given will be run.

Commands:
    all              Do all commands
    partition        Partition the live usb
    makefs-ext       Create the ext file system
    makefs-efi       Create the efi file system
    makefs           Both makefs-ext and makefs-efi
    copy-ext         Copy files to live usb ext partition
    copy-efi         Copy files to ESP partition
    copy             Both copy-ext and copy-efi
    uuids            Write UUIDs linking file systems
    install

Options:
  -c --clean        Delete files from each partition before copying
  -f --force        Ignore usb/removeable check (dangerous!)
  -F --force-ext    Force creation of ext4 filesystem even if one exists
  -g --gpt          Use gpt partitioning instead of msdos
  -h --help         Show this usage
  -L --label=Name   Label ext partition with Name
  -p --pretend      Don't run most commands, just show them
  -P --Pretend      Pretend witout verbose
  -q --quiet        Print less
  -s --size=XX      Percent of usb-device to use (default 100%)
  -v --verbose      Print more, show commands when run

Note: short options stack. Example: -Ff instead of -F -f
Note: options can be intermingled with commands and parameters
```

The Script
----------
The --verbose and --pretend (which implies --verbose) options were
meant to make it very clear what the script is doing.  They also
helped in debugging.  You can control which steps are done with
the commands.  The command "all" does everything.  There are a
few failsafes built in but they can be disabled with --force and
--force-ext.

Theory
------
We want to use ext4 for our LiveUSBs due to its ruggedness and
features.  But we need a fat32 partition in order to boot via UEFI.
So we use ext4 for the main LiveUSB partition and add a 2nd small
fat32 partition for booting via UEFI.  Legacy booting is done
normally with the ext4 partition.  The fat32 partition is only
used for UEFI booting.

Each partition needs to know about the other one.  We communicate
this with the UUIDs of the partitions.  The fat32 partition needs
to know where the kernel and initrd.gz are.  This is accomplished
with a line like:
```
search --no-floppy --set=root --fs-uuid $EXT4_UUID
```
in the grub.cfg file.

The Live system (on the ext4 partition needs to know about where
the Grub2 UEFI bootloader grub.cfg file is in order to be able
to save boot parameters selected by the user.  This is accomplished
with the antiX/esp-uuid file which contains the UUID of the
fat32 ESP partition.


Practice
--------
This program started out as proof-of-concept for passing along
instructions for creating ext4 LiveUSBs with a small fat32 partition
for UEFI booting.  In a nutshell: copy the efi/ and boot/ directories
to the fat32 partition and then record UUIDs so the two partitons know
about each other.

Legacy booting is done via the ext4 partition so it has the "boot"
flag set.  UEFI booting is done via the fat32 partition so it has the
"esp" flag set.  The fat32 partition only needs the contents of the
boot/ directory and the efi/ (or EFI/) directory.  Some of the
contents of boot/ is not needed but this only wastes a few Meg at
most.

One complication is the format of the grub.cfg file changed between
MX-15 and antiX-16.

For antiX-16, to specify the ext4 UUID you should uncomment the
following line and replace %UUID% with the UUID of the ext4
partition:
```
# search --no-floppy --set=root --fs-uuid %UUID%
```
For MX-15, you need to add the line.  This script adds it under the
"set menu_color_highlight" line which is not robust.  But starting
with MX-16 (or earlier) we will use a grub.cfg that is similar to the
one in antiX-16 so this non-robust approach is only for backward
compatibility with MX-15 and antiX-15.


Optimizing for Size
-------------------
I've tried to optimize the ext4 file system with these options:

```
m0 -N10000 -J size=32
```

This limits the number of inodes, the size of the journal and sets
aside no extra space reserved for root.  The idea is that our LiveUSB
does not normally contain many files (compared to an installed system)
nor does it need extra space reserved for root.  I've been using
"-N2000 -J size-16" for years without a problem.  I increased them by
factors of five and two for a greater margin of safety.  A user will
be limited to roughly 10,000 files on these LiveUSBs.  This seems like
a reasonable limit and yet provides significant savings in the
space overhead of the file system.

The partitioning is aligned on 1 MiB boundaries.
