---
title: "Synology Wireguard <samp>/dev/net/tun: no such file or directory</samp>"
date: 2024-12-24 15:00:00 -0800
categories: [Synology]
tags: [synology, wireguard, docker]
---

After updating the DSM version of my Synology NAS, I encountered the following error which prevented the `wg_hideme_privoxy` container from starting.

```
error gathering device information while adding custom device "/dev/net/tun": no such file or directory
```

[This](https://github.com/haugene/docker-transmission-openvpn/issues/1542#issuecomment-753022809) GitHub post helped fix the issue by manually creating the TUN device.

```shell
# Create the TUN device
mknod /dev/net/tun c 10 200
chmod 0755 /dev/net/tun
# Load the TUN module
insmod /lib/modules/tun.ko
```
