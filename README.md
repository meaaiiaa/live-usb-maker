# live-usb-maker
Create an antiX/MX LiveUSB
```
Usage: live-usb-maker [options] <iso-file> <usb-device> <commands>

Create a live-usb on <usb-device> from <iso-file>.  This will destroy
any existing information on <usb-device>.  Uses ext4 as the filesystem
for the live-usb and add a small fat32 file system for booting via UEFI.

  - Use "clone" as the iso-file to clone a running live system.
  - Use "clone=<dir>" to clone from a mounted live-usb or iso-file.
  - You may specify a device as the iso-file.  That device will be
    mounted and cloned.

At least one command must be given.  If "all" is not given then only the
commands given will be run.  Use a trailing "+" to run a command and all
commands after it.

Commands:
    sizes            Only show and check sizes, don't do anything
    all              Do all commands below
    partition        Partition the live usb
    makefs-ext       Create the ext file system
    makefs-fat       Create the fat file system
    makefs           Both makefs-ext and makefs-fat
    copy-ext         Copy files to live usb ext partition
    copy-fat         Copy files to fat partition
    copy             Both copy-ext and copy-fat
    uuids            Write UUIDs linking file systems
    cheats           Copy cheat codes to live-usb
    install          Install the legacy bootloader

Options:
  -a --auto         Never ask questions.  Always assume the safe answer
  -c --cheat=xxx    Add these cheatcodes to the live-usb
  -C --clear        Delete files from each partition before copying
  -d --debug        Pause before cleaning up
  -e --esp-size=XX  Size of ESP (fat) partition in MiB (default 50)
  -f --force=XXXX   Force the options specfied:
                        umount: Allows try to umount all partitions on drive
                           usb: Ignore usb/removable check
                        makefs: Make the ext4 filesystem even if one exists
                          copy: Overwrite ext4 partition even if antiX/ exists
                           all: All of the above (dangerous!)

  -g --gpt          Use gpt partitioning instead of msdos
  -h --help         Show this usage
  -L --label=Name   Label ext partition with Name
  -p --pretend      Don't run most commands, just show them
  -P --Pretend      Pretend witout verbose
  -q --quiet        Print less
  -s --size=XX      Percent of usb-device to use in (default 100)
  -v --verbose      Print more, show commands when run

Notes:
  - short options stack. Example: -Ff instead of -F -f
  - options can be intermingled with commands and parameters
  - config file: /root/.config/live-usb-maker/live-usb-maker.conf
  - the config file will be sourced if it exists
  - it will be created if it doesn't exist
```

The Script
----------
The `--verbose` and `--pretend` (which implies `--verbose`) options
were meant to make it very clear what the script is doing.  They also
helped in debugging.  You can control which steps are done with the
commands.  The command `all` does everything.  There are a few
failsafes built in but they can be disabled with `--force` and
`--force-ext`.

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
in the `grub.cfg` file.

The Live system (on the ext4 partition) needs to know about where
the Grub2 UEFI bootloader grub.cfg file is in order to be able
to save boot parameters selected by the user.  This is accomplished
with the `antiX/esp-uuid` file which contains the UUID of the
fat32 ESP partition.


Practice
--------
This program started out as proof-of-concept for passing along
instructions for creating ext4 LiveUSBs with a small fat32 partition
for UEFI booting.  In a nutshell: copy the `efi/` and `boot/`
directories to the fat32 partition and then record UUIDs so the two
partitons know about each other.

Legacy booting is done via the ext4 partition so it has the `boot`
flag set.  UEFI booting is done via the fat32 partition so it has the
`esp` flag set.  The fat32 partition only needs the contents of the
`boot/` directory and the `efi/` (or `EFI/`) directory.  Some of the
contents of boot/ is not needed but this only wastes a few Meg at
most.

One complication is the format of the grub.cfg file changed between
MX-15 and antiX-16.

For antiX-16, to specify the ext4 UUID you should uncomment the
following line and replace `%UUID%` with the UUID of the ext4
partition:
```
# search --no-floppy --set=root --fs-uuid %UUID%
```
For MX-15, you need to add the line.  This script adds it before
the first `menuentry` line.

Optimizing for Size
-------------------
I've tried to optimize the ext4 file system with these options:

```
-m0 -N10000 -J size=32
```

This limits the number of inodes, the size of the journal and sets
aside no extra space reserved for root.  The idea is that our LiveUSB
does not normally contain many files (compared to an installed system)
nor does it need extra space reserved for root.  I've been using
`-N2000 -J size-16` for years without a problem.  I increased them by
factors of five and two for a greater margin of safety.  A user will
be limited to roughly 10,000 files on these LiveUSBs.  This seems like
a reasonable limit and yet provides significant savings in the
space overhead of the file system.

The partitioning is aligned on 1 MiB boundaries.

Here are results from using various mkfs.ext4 options.  All sizes are
in Megabytes.  The savings are compared to the default settings.
These tests were all done on an 32-Gig Samsung Fit using the default
50 Meg fat32 partition in addition to the ext4 partition.

       mkfs.ext4 options        total  avail  savings
       ----------------------   -----  -----  -------
    A  -m0 -N2000  -J size=16   30522  30462     2116
    B  -m0 -N10000 -J size=32   30504  30444     2098
    C  -m1 -N10000 -J size=32   30504  30138     1792
    D  -m0                      29933  29873     1527
    E      -N10000 -J size=32   30504  28917      571
    F              -J size=32   30029  28422       76
    G                           29933  28346        0

Error Handling
--------------
Almost every system call is checked and if any fail it is a fatal
error.  With one exception all fatal errors will result in an exit
status of 100 and the error message being put into the file
`/var/log/live-usb-maker.error`. The one exception is the check for
root.  If that fails then we can't write to the error log anway so
in that case we return an exit status of 99.  It's easy to make this
more complicated (for example use a different status for problems with
reading the command line parameters).

### Error Codes
Some errors also create and error code and a question in the error
log file.  The format is:

    <code>:<message>
    Q:<question>

If you are calling this code from a GUI then you can get around these
errors by reporting the <message> as a warning/non-fatal error and
then asking the user the <question>.  If the answer "no" then you must
exit.  If they answer use then call the program again and add the
parameter:

    --force=<code>

If this happens multiple times then the codes can accumulate:

    --force=<code1>,<code2>,...

The codes will always contain only lowercase letters and possible
hyphens.   For most errors, there won't be a code and a question so
the only thing in the error log will be:

    :<message>

You are, of course, free to ignore the code and the question and just
treat the message like a fatal error.  Here are the 3 messages that
use this mechanism:

    flock:The flock program was not found.
    Q:Do you want to continue without locking

    umount:One or more partitions on device %s are mounted at:  %s
    Q:Do you want these partitions umounted

    usb:The device %s does not seem to be usb or removeable
    Q:To you want to use it anyway (dangerous!)

Of course, there is a possible race condition with the flock error
message.  One way around this would the to use an exit status for the
message instead of the error log but that would mean the error message
and the question would need to be hard-coded into the GUI program.
