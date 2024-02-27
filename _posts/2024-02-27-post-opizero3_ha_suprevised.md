---
title: "Post: Home Assistant Supervised on Orange Pi Zero3."
last_modified_at: 2024-02-27T22:02:02-05:00
categories:
  - Blog
tags:
  - Post Formats
  - readability
  - standard
---

## Home Assistant Supervised on Orange Pi Zero3

Download image:
```
https://github.com/armbian/community/releases/download/24.2.0-trunk.540/Armbian_community_24.2.0-trunk.540_Orangepizero3_bookworm_current_6.6.16.img.xz
```
Install image to SD (balenaEther)

Boot and configure you new installed system (network, hostname, updates, etc)
Reboot if required.
```
ambrian-config
...
```
All steps as root user.
```
apt install apparmor
echo "extraargs=apparmor=1 security=apparmor systemd.unified_cgroup_hierarchy=false systemd.legacy_systemd_cgroup_controller=false" >> /boot/armbianEnv.txt
update-initramfs -u
reboot
```
We install dependencies
```
apt-get install jq wget curl avahi-daemon udisks2 libglib2.0-bin network-manager dbus systemd-journal-remote cifs-utils -y
```
If any apt problems:
```
apt --fix-broken install
```
Install HA os-agent:
```
nano /etc/os-release
```
Replace first line, must be:
```
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
...
```
Downloadand install os-agent:
```
wget https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_aarch64.deb
dpkg -i os-agent_1.6.0_linux_aarch64.deb
```
Install Docker: 
```
sudo curl -fsSL get.docker.com | sh
```
Install HA supervised:
```
wget https://github.com/home-assistant/supervised-installer/releases/download/1.7.0/homeassistant-supervised.deb
dpkg -i homeassistant-supervised.deb
```
Select qemuarm-64

Wait and look when we have service listen on 8123
```
ss -tulpn| grep 8123
```
Login to you HA webUI
```
https://192.168.1.xxx:8123
```
Migration from old HA. 
You need preinstall all addon's if you migrate like the previous one before full backup restore.
You do not need to configure them, just install it.
Additional steps: 
1. Addon "Mosquitto broker"
2. Addon "Terminal & SSH"
3. HACS
run Terminal
wget -O - https://get.hacs.xyz | bash -
Then follow by https://hacs.xyz/docs/configuration/basic
3. Zigbee2mqtt: https://github.com/zigbee2mqtt/hassio-zigbee2mqtt#installation
4. Additional Addon's
...
5. Upload fullbackup
6. Restore fullbackup

Remove grap (all conteiners (addon's, etc) must be running before you do that)
Don't do it if you don't know what it does:
```
docker system prune -a
```
