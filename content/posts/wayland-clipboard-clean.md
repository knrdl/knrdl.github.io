---
title: "Wayland: Auto-clean clipboard"
date: "2024-01-04"
tags: [wayland, clipboard, bash]
lang: "en"
---

Wayland has the safety feature of clearing the clipboard if you close the source window (where the content was copied from). Password managers (like KeePass) also give you only a few seconds to paste the password somewhere before clearing the clipboard. But in day-to-day life, you might need to copy secrets and tokens outside a password manager. To prevent accidental pastes, these values should not stay in the clipboard forever. A manual attempt is to overwrite the clipboard by copying something harmless afterwards, but that's not very fault-tolerant. The automatic solution is to clear the clipboard if the last copy action is older than a certain delay.

```shell
sudo apt install wl-clipboard
```

Executable file `/usr/local/bin/clipboard-cleaner-handler`:
```shell
#!/bin/dash

# remove previous delay handlers
pgrep -f -x "/bin/dash $0" | grep -v ^$$\$ | while read pid ; do 
    kill $pid
done

sleep 45  # <- the delay
wl-copy -c
echo cleared clipboard
```

Executable file `/usr/local/bin/clipboard-cleaner`:
```shell
#!/bin/dash
wl-paste -w /bin/dash -c '/usr/local/bin/clipboard-cleaner-handler &'
```

Finally put `clipboard-cleaner` into autostart. 


How does it work: `wl-paste` watches for clipboard copy events and executes `clipboard-cleaner-handler` each time without blocking the listening. `clipboard-cleaner-handler` first removes all invocations of itself from previous copy events. Then it waits for the delay before cleaning the clipboard. That way, only the last triggered delay actually counts.
