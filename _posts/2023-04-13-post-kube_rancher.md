---
title: "Post: Kubernetes node for Rancher on Debian 11 VM."
last_modified_at: 2023-04-13T22:02:02-05:00
categories:
  - Blog
tags:
  - Post Formats
  - readability
  - standard
---

## Kubernetes node for Rancher on Debian 11.

Paigaldus Debian 11'le

Swap'i ei loo nii master'il, kui worker'il


Master
```
/               20G
/var            20G
/var/lib/etcd   1G
```
Worker
```
/               20G
/var            32-100G
```

Vajalikud draiverid:
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
````
```
sysctl --system
```
Keelame IPv6'e:
```
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
```
Master'il eemaldame etcd jaoks "lost+found"
```
rm -rf  /var/lib/etcd/*
```
Vajalikud paketid
```
apt install net-tools dnsutils qemu-guest-agent tcpdump man apt-transport-https ca-certificates curl gnupg-agent software-properties-common sudo chrony apparmor apparmor-utils
```
SSH võti root kasutajale:
```
ssh localhost
```
ning katkestad...(CTRL+C) luuakse .ssh
```
#nano  .ssh/authorized_keys
echo "ssh-ed25519 you_public_key" >> .ssh/authorized_keys
````
Õigused paika:
```
chmod 600 .ssh/authorized_keys
```
Staatiline IP (jätta ka alles ning vajadusel seadistada ka DHCP's server)
```
nano /etc/network/interfaces
#iface ens18 inet dhcp
iface ens18 inet static
        address 192.168.1.xxx/24
        gateway 192.168.1.1
```
DNS
```
cat <<EOF | tee /etc/resolv.conf
domain    example.com
search    example.com
nameserver  192.168.1.1
nameserver  192.168.1.101
nameserver  192.168.1.100
options timeout:2
EOF
```

Bash värviliseks:
```
cp /etc/skel/.bashrc .
sed -i 's/01;32m/01;31m/g' .bashrc
```

Host'i nime seadmine
```
hostnamectl set-hostname node_name
```
```
echo -e "LANG=en_US.UTF-8\nLC_ALL=C.UTF-8" > /etc/default/locale
````
```
dpkg-reconfigure locales
```
```
reboot
```
Kustutame ajutise kasutaja:
```
deluser --remove-home user
```
Docker'i paigaldus
Lisame repo:
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
```
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Paigaldame Docker'i \
Kontrollime, mis versioonid saadaval
```
apt update
apt-cache madison docker-ce
```
Paigaldame viimase versiooni
```
apt-get install docker-ce docker-ce-cli containerd.io
```
Paketid hold'i
```
apt-mark hold docker-ce docker-ce-cli
```
Draiveri tuuning
```
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
Edasi vajaliku käsu'd node liitmise jaoks Ransher'isse saad rancherist. Vaata, et etcd ja controlplane master'ile ning worker'ile ainult worker:\
(a la master)
```
docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.7.1 --server https://192.168.1.102 --token zf2ql45sltvds7dft5p2k4c5hnnqdtvcwn --ca-checksum fe28b2a87ec6e72211f56ea5154c7770c6ce5c2 --etcd --controlplane
```
(a la worker)
```
docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  rancher/rancher-agent:v2.7.1 --server https://192.168.1.102 --token zf2ql45sltvds7dft5p2k4c5hnnqdtvcwn --ca-checksum fe28b2a87ec6e72211f56ea5154c7770c6ce5c2 --worker
```