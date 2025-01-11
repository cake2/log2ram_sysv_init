# log2ram_sysv_init
log2ram for sysv
The below is from https://energytalk.co.za/t/gx-device-logging-to-ram-tmpfs-log2ram/2130/2
I modified it for Devuan (sysv init)
I will pretty this all up later.

mkdir -p /data/my_files
cd /data/my_files
curl -L https://github.com/azlux/log2ram/archive/master.tar.gz | tar zxf -
cd log2ram-master
```
mkdir -p /data/log2ram_system
cp log2ram /data/log2ram_system/log2ram
chmod 755 /data/log2ram_system/log2ram
cp log2ram.conf /data/log2ram_system/log2ram.conf
chmod 644 /data/log2ram_system/log2ram.conf
cp log2ram.logrotate /data/log2ram_system/log2ram.logrotate
chmod 644 /data/log2ram_system/log2ram.logrotate
```
nano /data/log2ram_system/log2ram-init.sh
```
#!/bin/sh
#
# log2ram SysV Init script
# Developed by kivanov for VenusOS
#
### BEGIN INIT INFO
# Provides:          log2ram
# Required-Start:    $local_fs 
# Required-Stop:        
# Default-Start:     S
# Default-Stop:      0 6
# Short-Description: Provides ramdrive for system logging
### END INIT INFO

# Init start
START=06
PATH="/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"
LOG2RAM_SCRIPT=/usr/local/bin/log2ram
CONFIG=/etc/log2ram.conf

# Check if all is fine with the needed files
[ -f $CONFIG ] || exit 1
[ -x $LOG2RAM_SCRIPT ] || exit 1

# Source function library.
if [ -f /etc/init.d/functions ]; then
      . /etc/init.d/functions
elif [ -f /etc/rc.d/init.d/functions ]; then
      . /etc/rc.d/init.d/functions
fi


case "$1" in
  start)
    echo -n "Starting log2ram: "
    #touch /data/log2ram.started.$(date +"%Y-%m-%d_%T")
    ${LOG2RAM_SCRIPT} start
    RETVAL=$?
    if [ $RETVAL -eq 0 ] ; then
        echo "OK"
    else
        echo "FAIL"
    fi
    ;;
  stop)
    echo -n "Stopping log2ram: "
    #touch /data/log2ram.stopped.$(date +"%Y-%m-%d_%T")
    ${LOG2RAM_SCRIPT} stop
    RETVAL=$?
    if [ $RETVAL -eq 0 ] ; then
        echo "OK"
    else
        echo "FAIL"
    fi
    ;;
  sync)
    echo -n "This operation is generally provided by cron."
    while true; do
    read -p "Continue (y/n)?" choice
    case ${choice} in
        [Yy]* ) break;;
        [Nn]* ) exit 1;;
        * ) echo "Please answer yes or no.";;
    esac
        done

    echo -n "Force log2ram write to disk on-the-fly from the cli: "
    #touch /data/log2ram.synched.$(date +"%Y-%m-%d_%T")
    ${LOG2RAM_SCRIPT} write
    RETVAL=$?
    if [ $RETVAL -eq 0 ] ; then
        echo "OK"
    else
        echo "FAIL"
    fi
    ;;
  status)
    cat /proc/mounts | grep -w log2ram > /dev/null && { echo -ne "log2ram is running\n"; };
    cat /proc/mounts | grep -w log2ram > /dev/null || { echo -ne "log2ram is NOT running\n"; };
    exit $?
    ;;
  restart)
    $0 stop && sleep 1 && $0 start
    ;;
  *)
    echo "Usage: /etc/init.d/$(basename $0) {start|stop|sync|status|restart}"
    exit 1
esac

exit 0
```


chmod 755 /data/log2ram_system/log2ram-init.sh

nano /data/log2ram_system/log2ram-cron.sh
```
#!/bin/sh
#
# cron.daily/log2ram -- daily write/sync ramlog to disk
#
#
LOG2RAM_SCRIPT=/usr/local/bin/log2ram
CONFIG=/etc/log2ram.conf
# Check if all is fine with the needed files
[ -f $CONFIG ] || exit 1
[ -x $LOG2RAM_SCRIPT ] || exit 1
cat /proc/mounts | grep -w log2ram > /dev/null || { echo -ne "log2ram is NOT running\n"; exit 1; }

exec ${LOG2RAM_SCRIPT} write
```
chown -R root:root /data/log2ram_system

change /data/log2ram_system/log2ram.conf
```
SIZE=128M
USE_RSYNC=true
MAIL=false
PATH_DISK="/var/log"
ZL2R=false
```
apt install rsync util-linux
```
cd /data/log2ram_system
mkdir -p /usr/local/bin/
cp log2ram /usr/local/bin/log2ram
cp log2ram.conf /etc/log2ram.conf
cp log2ram-init.sh /etc/init.d/log2ram-init.sh
```
we create daily cron for syncing
```
###remove periods and dashes for filenames inside /etc/cron.daily/
cp log2ram-cron.sh /etc/cron.daily/log2ram
[ -d /etc/logrotate.d ] && cp /data/log2ram_system/log2ram.logrotate /etc/logrotate.d/log2ram

update-rc.d log2ram-init.sh start 06 S . stop 80 0 6 .
```
Checking correct symlinks are installed:
```
ls -alh /etc/rcS.d/    # symlink should be present
ls -alh /etc/rc3.d/    # symlink should NOT be present
ls -alh /etc/rc6.d/    # symlink should be present
```

sync
reboot
