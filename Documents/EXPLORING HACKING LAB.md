## How to set up lab

How to set the network connectio to surviving reboot

> on kali,
```
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.77.77.10/24 \
  ipv4.gateway "" \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.autoconnect yes
sudo nmcli connection up "Wired connection 1"

```


> on Ubuntu server
```
sudo nano /etc/netplan/00-installer-config.yaml

network:
  version: 2
  ethernets:
    enp1s0:
      match:
        macaddress: "52:54:00:b2:73:de"
      set-name: enp1s0
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.77.77.20/24
          

sudo netplan generate

sudo netplan retry
```


## Attacking the victim machine

## CSFR

