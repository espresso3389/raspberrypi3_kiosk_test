## apt

Update `/etc/apt/source.list` to enable faster download in Japan:

```
deb http://ftp.jaist.ac.jp/raspbian/ stretch main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ stretch main contrib non-free rpi
```

## Install certain packages

```
sudo apt-get install unclutter mesa-vdpau-drivers
```

## chromium-browser

On `chrome://flags`, enable the following parameters:

```
#enable-gpu-rasterization
#enable-zero-copy
```

## Kiosk GUI

`~/.config/lxsession/LXDE-pi/autostart`:

```
@xset s off
@xset -dpms
@xset s noblank

@unclutter
@chromium-browser --enable-native-gpu-memory-buffers --incognito --kiosk http://example.com
```

# Make SD card almost read-only using overlayFS

[Setting up overlayFS on Raspberry Pi](https://www.domoticz.com/wiki/Setting_up_overlayFS_on_Raspberry_Pi)

# rw/ro

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
