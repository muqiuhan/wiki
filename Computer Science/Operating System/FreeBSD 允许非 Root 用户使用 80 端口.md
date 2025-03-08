#freebsd #os

In experimenting with FreeNAS jails I wanted to allow a web service to use port 80. Normally 80 is a high order port reserved for root-level processes for security reasons. Since this is a FreeBSD jail and not a full on system I’m not worried about this.

The command to do so is fairly simple (thanks to [this page](http://hyber.org/privbind.yaws) for information)

`sysctl net.inet.ip.portrange.reservedhigh=0`

The above command is not permanent; to make it so add it to `/etc/sysctl.conf`:

`echo "net.inet.ip.portrange.reservedhigh=0" >> /etc/sysctl.conf`