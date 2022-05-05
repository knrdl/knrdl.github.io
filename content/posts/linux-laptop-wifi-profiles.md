---
title: "Linux Laptop: Wifi dependent profiles"
date: "2022-05-05"
tags: ["linux", "network-manager", "wifi", "bash"]
lang: "en"
hackerNewsId: ""
---

A notebook as mobile device may should behave differently depending on the location. Examples:

* at home:
    * sound output should be turned on
    * a backup script can run from time to time
* in a foreign network / without connection:
    * speakers should be muted
    * a vpn connection should be established in public networks
* at work:
    * speakers should be muted
    * time tracking software should be active

Evaluating the ssid (name) of the currently connected Wi-Fi on connection change is a cheap solution. This can be done
by using a hook of the network manager.

1. Create file `/etc/NetworkManager/dispatcher.d/90-wifi-profiles` (replace 3x `yourname`)

```shell
#!/usr/bin/env bash

interface=$1
event=$2

# only listen to connect/disconnect events
[[ "$event" == "connectivity-change" ]] || exit 0   

# check user is logged in
who | awk '{print $1}' | grep -q '^yourname$' || exit 0

sudo -u yourname /home/yourname/.wifi-profile >> /var/log/wifi-profiles 2>&1
```

and run

```shell
$ sudo chown root:root /etc/NetworkManager/dispatcher.d/90-wifi-profiles
$ sudo chmod o+x /etc/NetworkManager/dispatcher.d/90-wifi-profiles
```

2. Create `~/.wifi-profile`

```shell
#!/usr/bin/env bash

# user specific env vars need to be set manually!
export XDG_RUNTIME_DIR=/run/user/1000
export DISPLAY=:0

function soundOff() {
  amixer -D pulse sset Master mute
}

function soundOn() {
  amixer -D pulse sset Master 40%
  amixer -D pulse sset Master unmute
}

function askConnectVpn() {
  [[ zenity --question --title "Foreign Wifi" --text "Connect to home VPN?" ]] && nmcli con up id yourvpn
}

SSID="$(iwgetid -r)"

if [[ "$SSID" = 'home_network1' || "$SSID" = 'home_network2' ]]; then  # at home
  soundOn  # use carefully, might not be wanted
  # run backup
elif [[ "$SSID" = 'work_network1' || "$SSID" = 'work_network2' ]]; then  # at work
  soundOff
elif [[ "$SSID" = '' ]]; then  # not connected
  soundOff
else  # foreign network
  soundOff
  askConnectVpn
fi
```

and run

```shell
$ chmod o+x ~/.wifi-profile
```

3. Test

```shell
$ sudo /etc/NetworkManager/dispatcher.d/90-wifi-profiles wlan0 connectivity-change
$ cat /var/log/wifi-profiles
```
