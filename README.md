## apt Configuration

Update `/etc/apt/source.list` to enable faster download in Japan:

```
deb http://ftp.jaist.ac.jp/raspbian/ stretch main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ stretch main contrib non-free rpi
```

## Full-HD video playback on chromium

### Prerequisites

`mesa-vdpau-drivers` is required to realize smooth Full-HD video playback on chromium.

```
sudo apt-get install mesa-vdpau-drivers
```

### raspi-config

Launch raspi-config and change GPU memory to 128MB on \[7 Advanced Options\] - \[A3 Memory Split\].

```
sudo raspi-config
```

### chromium-browser

On `chrome://flags`, enable the following parameters:

```
#enable-gpu-rasterization
#enable-zero-copy
```

You should launch `chromium-browser` with `--enable-native-gpu-memory-buffers` option to really activate these options; see the actual command line below.

## Kiosk GUI

`~/.config/lxsession/LXDE-pi/autostart`:

```sh
@xset s off
@xset -dpms
@xset s noblank

@chromium-browser --enable-native-gpu-memory-buffers --incognito --kiosk file:///home/pi/somewhere/kiosk.html

/home/pi/mousetrick.sh
```

And `/home/pi/mousetrick.sh` is like the following:

```sh
#!/bin/sh

sleep 8
xdotool mousemove 0 0
sleep 1
xdotool mousemove 1920 1080
```

The file is used to hide mouse on chromium-browser; even if we use `cursor: none` on element style, chromium does not hide the cursor unless mouse moves. The actual sleep duration may vary according to the HTML contents.

## Make SD card almost read-only using overlayFS

The instruction is based on the following article: [Setting up overlayFS on Raspberry Pi](https://www.domoticz.com/wiki/Setting_up_overlayFS_on_Raspberry_Pi)

### Download mount_overlay/saveoverlays

Download mount_overlay/saveoverlays and customize them:

#### /usr/local/bin/mount_overlay

```bash
#!/bin/bash
DIR="$1"
[ -z "${DIR}" ] && exit 1
#if ! grep -q overlay /proc/filesystems
#then
#    echo "Filesystem overlay is not available. You need a kernel update: apt-get update && apt-get upgrade"  >&2
#    exit 2
#fi
if [ ! -d "${DIR}_org" ]
then
    echo "${DIR}_org does not exist" >&2
    exit 1
fi
if [ ! -d "${DIR}_rw" ]
then
    echo "${DIR}_rw does not exist" >&2
    exit 1
fi
#
# ro must be the first mount option for root .....
#
ROOT_MOUNT=$( awk '$2=="/" { print substr($4,1,2) }' /proc/mounts )
if [ "$ROOT_MOUNT" != "ro" ]; then
    /bin/mount --bind ${DIR}_org ${DIR}
else
    /bin/mount -t tmpfs ramdisk ${DIR}_rw
    /bin/mkdir ${DIR}_rw/upper
    /bin/mkdir ${DIR}_rw/work
    OPTS="-o lowerdir=${DIR}_org,upperdir=${DIR}_rw/upper,workdir=${DIR}_rw/work"
    /bin/mount -t overlay ${OPTS} overlay ${DIR}
fi
```

#### /etc/init.d/saveoverlays

```bash
#!/bin/bash
### BEGIN INIT INFO
# Provides:          saveoverlays
# Required-Start:    $local_fs $time
# Required-Stop:     umountfs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Manage sync of overlay mounts
# Description:       Manage sync of overlay mounts
### END INIT INFO

# Do NOT "set -e"
#
# Testing hints:
#       sudo mount -o remount,ro /
#       sudo env INIT_VERBOSE=yes /etc/init.d/saveoverlays stop
#       cat /var/log/saveoverlays.log

PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
DESC="overlay filesystem sync"
NAME=saveoverlays

if [ -f "/etc/default/$NAME" ]; then
    . "/etc/default/$NAME"
fi
TMPLOG=/tmp/$NAME.log
TESTDIR=/var/spool/cron/crontabs
LOGFILE=/var_org/log/$NAME.log
ROOTLOG="${HOME}/$NAME.log"

OVERLAYFS=${OVERLAYFS:-$(  awk '/^overlay/ { print $2 }' /proc/mounts )}
SYNCEXCLUDES=${SYNCEXCLUDES:-'--exclude *.leases'}
SYNCFLAGS=${SYNCFLAGS:-"-avH --delete-after --inplace --no-whole-file"}
KILLPROCS=${KILLPROCS:-"domoticz mosquitto"}

# Check if we are running with read-only root
# ROROOT=$( mount | egrep '^/dev/.*on / .*ro,' )
ROROOT=$( awk '$2=="/" { print substr($4,1,2) }' /proc/mounts )
DOSYNC=${FORCESYNC:-"$ROROOT"}

# Load the VERBOSE setting and other rcS variables
. /lib/init/vars.sh

# Define LSB log_* functions.
# Depend on lsb-base (>= 3.2-14) to ensure that this file is present
# and status_of_proc is working.
. /lib/lsb/init-functions

#
# Function that starts the daemon/service
#
function do_start() {
        if [ "${ROROOT}" = "ro" ]
        then
                log_action_msg "Read-only root active"
        else
                log_action_msg "Read-only root inactive"
        fi
        if date '+%Y' | grep -q 1970
        then
                log_action_msg "Clock is not set. Trying fake-hwclock"
                fake-hwclock load
        fi
}

#
# Function that syncs the files
#
function do_sync() {
        RETVAL=0
        if [ ! -d "${TESTDIR}" ]
        then
                log_action_msg "$DESC disabled. Cannot find $TESTDIR. Check on start/stop order"
                RETVAL=2
        elif [ "${ROROOT}" != "ro" ] || mount -o remount,rw / >> ${TMPLOG} 2>&1
        then
                #
                # If we run with overlayfs, try to sync the dirs
                #
                echo "----------------------------"             >> ${TMPLOG}
                echo "$NAME sync started at $( date )" >> ${TMPLOG}
                if [ -n "${KILLPROCS}" ]
                then
                        for P in ${KILLPROCS}
                        do
                                for S in 3 5 5
                                do
                                        PID=$( pgrep -x "$P" )
                                        [ -z "${PID}" ] && break
                                        echo "Found running $P - killing ${PID} ..."
                                        kill "${PID}"
                                        sleep ${S}
                                done
                        done
                        KILLPROCS=""
                fi
                for DIR in ${OVERLAYFS}
                do
                        SOURCE="${DIR}"
                        STAGE="${DIR}_stage"
                        DEST="${DIR}_org"
                        log_action_msg "Syncing ${SOURCE}..."
                        echo "----"                                      >> ${TMPLOG}
                        echo "$NAME sync ${SOURCE} to ${DEST} with options ${SYNCFLAGS} ${SYNCEXCLUDES} at $( date )" >> ${TMPLOG}
                        if [ -d "${SOURCE}" -a -d "${DEST}" ] && mkdir -p "${STAGE}" >> ${TMPLOG} 2>&1
                        then
                                echo "---- Staging to $STAGE ------"                    >> ${TMPLOG}
                                rsync ${SYNCFLAGS} ${SYNCEXCLUDES} ${SOURCE}/ ${STAGE}/ >> ${TMPLOG} 2>&1
                                # echo "---- Unmounting ${SOURCE} ------"                 >> ${TMPLOG}
                                # umount "${SOURCE}"                                      >> ${TMPLOG}
                                echo "---- Copy to $DEST ------"                        >> ${TMPLOG}
                                rsync ${SYNCFLAGS} ${SYNCEXCLUDES} ${STAGE}/  ${DEST}/  >> ${TMPLOG} 2>&1
                        else
                                log_action_msg "Skipping this step: ${SOURCE} or ${DEST} or ${STAGE} not available"
                        fi
                        echo "$NAME sync ${SOURCE} to ${DEST} ended at $( date )" >> ${TMPLOG}
                done
                cat ${TMPLOG} >> ${LOGFILE}
                [ -f ${ROOTLOG}.2 ] && cp ${ROOTLOG}.2 ${ROOTLOG}.3
                [ -f ${ROOTLOG}.1 ] && cp ${ROOTLOG}.1 ${ROOTLOG}.2
                [ -f ${ROOTLOG}   ] && cp ${ROOTLOG}   ${ROOTLOG}.1
                cp ${TMPLOG} ${ROOTLOG}
                if [ -w /etc/fake-hwclock.data ]
                then
                        log_action_msg "Saving fake-hwclock"
                        fake-hwclock save
                fi
                log_action_msg "Sync changes to disk"
                sync; sync; sync
                #
                # return to read-only only if that is where we started
                #
                if [ "${ROROOT}" == "ro" ]
                then
                        log_action_msg "Remount root as read-only"
                        mount -o remount,ro /
                fi
        else
                log_action_msg "Remounting root as writeable failed!"
                RETVAL=2
        fi
        return "$RETVAL"
}

#
# Function that stops the daemon/service
#
function do_stop() {
        #
        # If we run with overlayfs, try to sync the dirs
        #
        if [ -n "${DOSYNC}" ]
        then
                do_sync;
                return $?
        else
                log_action_msg "Root is not read-only. No action"
                return 0
        fi
}


case "$1" in
  start)
        log_daemon_msg "Starting $DESC" "$NAME"
        do_start
        case "$?" in
                0|1) log_end_msg 0 ;;
                2)   log_end_msg 1 ;;
        esac
        ;;
  stop)
        log_daemon_msg "Stopping $DESC" "$NAME"
        do_stop
        case "$?" in
                0|1) log_end_msg 0 ;;
                2)   log_end_msg 1 ;;
        esac
        ;;
  sync)
        log_daemon_msg "Syncing $DESC" "$NAME"
        do_sync
        case "$?" in
                0|1) log_end_msg 0 ;;
                2)   log_end_msg 1 ;;
        esac
        ;;
  status)
        ;;
  restart)
        ;;
  *)
        echo "Usage: $SCRIPTNAME {start|stop|status|restart}" >&2
        exit 3
        ;;
esac
```

### Backup and recovery

```sh
sudo cp -v /boot/cmdline.txt /boot/cmdline.txt-orig
sudo cp -v /etc/fstab /etc/fstab-orig
cd /var
sudo tar --exclude=swap -cf /var-orig.tar .
```

### Stop swap

```sh
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall 
sudo update-rc.d dphys-swapfile disable
sudo systemctl disable dphys-swapfile
```

### Move certain files

```sh
sudo service fake-hwclock stop 
sudo mv /etc/fake-hwclock.data /var/log/fake-hwclock.data
sudo ln -s /var/log/fake-hwclock.data /etc/fake-hwclock.data
sudo service fake-hwclock start
sudo update-rc.d fake-hwclock disable
sudo systemctl disable fake-hwclock
```
 
### Set up fuse and mount script
 
```sh
sudo apt-get install fuse lsof
```
 
### Set up the saveoverlays service
 
```sh
sudo systemctl enable saveoverlays.service
```
 
### Change boot command line

On `/boot/cmdline.txt`, add `noswap fastboot ro` to the end of line like this (actual content is different on each machine):

```
dwc_otg.lpm_enable=0 console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait noswap fastboot ro
```

### Update /etc/fstab for read-only SD card

`/etc/fstab` need to be updated similar to this:

```
proc            /proc           proc    defaults    0 0
/dev/mmcblk0p1  /boot           vfat    ro          0 2
/dev/mmcblk0p2  /               ext4    ro,noatime  0 1
mount_overlay   /var            fuse    nofail,defaults,x-systemd.automount,x-systemd.requires=/var,x-systemd.device-timeout=10s 0 0
mount_overlay   /home           fuse    nofail,defaults,x-systemd.automount,x-systemd.requires=/home,x-systemd.device-timeout=10s 0 0
none            /tmp            tmpfs   defaults    0 0
```

### Prepare the overlay directories

```sh
sudo service ntp stop
sudo service rsyslog stop

sudo mv /home /home_org
sudo mkdir /home /home_rw 
sudo mv /var /var_org
sudo mkdir /var /var_rw
```

### Loopback mount /home and /var before boot

This is to make sure the next reboot does not run into trouble due to moved data (`rsync` with delete). When root is not read-only, this will result in `/var_org` being loopback mounted to `/var`, and `/home_orig` loopback mounted to `/home`

```sh
sudo mount /home
sudo mount /var
```

### Reboot

```sh
sudo reboot
```

## rw/ro command to switch read-write/read-only of root file system

`/etc/bash.bashrc`:

```
# set variable identifying the filesystem you work in (used in the prompt below)
set_bash_prompt(){
    fs_mode=$(mount | sed -n -e "s/^\/dev\/.* on \/ .*(\(r[w|o]\).*/\1/p")
    PS1='\[\033[01;32m\]\u@\h${fs_mode:+($fs_mode)}\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
}

alias ro='sudo service saveoverlays sync ; sudo mount -o remount,ro / ; sudo mount -o remount,ro /boot'
alias rw='sudo mount -o remount,rw / ; sudo mount -o remount,rw /boot'

# setup fancy prompt"
PROMPT_COMMAND=set_bash_prompt
```

`/etc/bash.bash_logout`:

```
sudo service saveoverlays sync
sudo mount -o remount,ro /
sudo mount -o remount,ro /boot
```

The code is based on the following article: [Protect your Raspberry PI SD card, use Read-Only filesystem](https://hallard.me/raspberry-pi-read-only/)
