#linux #os

The udevd daemon operates as follows:
1. The kernel sends udevd a notification event, called a uevent, through an internal network link.
2. udevd loads all of the attributes in the uevent.
3. udevd parses its rules, filters and updates the uevent based on
those rules, and takes actions or sets more attributes accordingly. An incoming uevent that udevd receives from the kernel might look like this:

```
ACTION=change
DEVNAME=sde
DEVPATH=/devices/pci0000:00/0000:00:1a.0/usb1/1-1/1-1.2/1-1.2:1.0/host4/
target4:0:0/4:0:0:3/block/sde
DEVTYPE=disk
DISK_MEDIA_CHANGE=1
MAJOR=8
MINOR=64
SEQNUM=2752
SUBSYSTEM=block
UDEV_LOG=3
```

This particular event is a change to a device. After receiving the uevent, udevd knows the name of the device, the sysfs device path, and a number of other attributes associated with the properties; it is now ready to start processing rules.
The rules files are in the /lib/udev/rules.d and /etc/udev/rules.d directories. The rules in /lib are the defaults, and the rules in /etc are overrides. A full explanation of the rules would be tedious, and you can learn much more from the udev(7) manual page, but here is some basic information about how udevd reads them:
1. udevd reads rules from start to finish of a rules file.
2. After reading a rule and possibly executing its action, udevd
continues reading the current rules file for more applicable rules.
3. There are directives (such as GOTO) to skip over parts of rules files
if necessary. These are usually placed at the top of a rules file to skip over the entire file if it’s irrelevant to a particular device that udevd is configuring.

Let’s look at the symbolic links from the /dev/sda example. Those links were defined by rules in /lib/udev/rules.d/60-persistent-storage.rules. Inside, you’ll see the following lines:
```
# ATA
KERNEL=="sd*[!0-9]|sr*", ENV{ID_SERIAL}!="?*", SUBSYSTEMS=="scsi",
ATTRS{vendor}=="ATA", IMPORT{program}="ata_id --export $devnode"
# ATAPI devices (SPC-3 or later)
KERNEL=="sd*[!0-9]|sr*", ENV{ID_SERIAL}!="?*", SUBSYSTEMS=="scsi",
ATTRS{type}=="5",ATTRS{scsi_level}=="[6-9]*", IMPORT{program}="ata_id --
export $devnode"
```
These rules match ATA disks and optical media presented through the kernel’s SCSI subsystem. You can see that there are a few rules to catch different ways the devices may be represented, but the idea is that udevd will try to match a device starting with sd or sr but without a number (with the `KERNEL =="sd*[!0-9]|sr*"` expression), as well as a subsystem (`SUBSYSTEMS=="scsi"`), and, finally, some other attributes, depending on the type of device. If all of those conditional expressions are true in either of the rules, udevd moves to the next and final expression:
```
IMPORT{program}="ata_id --export $tempnode"
```
This is not a conditional. Instead, it’s a directive to import variables from the /lib/udev/ata_id command. If you have such a disk, try it yourself on the command line. It will look like this:
```
# /lib/udev/ata_id --export /dev/sda
ID_ATA=1
ID_TYPE=disk
ID_BUS=ata
ID_MODEL=WDC_WD3200AAJS-22L7A0
ID_MODEL_ENC=WDC\x20WD3200AAJS22L7A0\x20\x20\x20\x20\x20\x20\x20\x20\x20
\x20
\x20\x20\x20\x20\x20\x20\x20\x20\x20
ID_REVISION=01.03E10
ID_SERIAL=WDC_WD3200AAJS-22L7A0_WD-WMAV2FU80671
--snip--
```
The import now sets the environment so that all of the variable names in this output are set to the values shown. For example, any rule that follows will now recognize ENV{ID_TYPE} as disk. In the two rules we’ve seen so far, of particular note is ID_SERIAL. In each rule, this conditional appears second:
```
ENV{ID_SERIAL}!="?*"
```
This expression evaluates to true if ID_SERIAL is not set. Therefore, if ID_SERIAL is set, the conditional is false, the entire current rule does not apply, and udevd moves to the next rule.
Why is this here? The purpose of these two rules is to run ata_id to find the serial number of the disk device and then add these attributes to the current working copy of the uevent. You’ll find this general pattern in many udev rules. With `ENV{ID_SERIAL}` set, udevd can now evaluate this rule later on in the rules file, which looks for any attached SCSI disks:
```
KERNEL=="sd*|sr*|cciss*", ENV{DEVTYPE}=="disk", ENV{ID_SERIAL}=="?
*",SYMLINK+="disk/by-id/$env{ID_BUS}-$env{ID_SERIAL}"
```
You can see that this rule requires `ENV{ID_SERIAL}` to be set, and it has one directive:
```
SYMLINK+="disk/by-id/$env{ID_BUS}-$env{ID_SERIAL}"
```

This directive tells udevd to add a symbolic link for the incoming device. So now you know where the device symbolic links came from! You may be wondering how to tell a conditional expression from a directive.