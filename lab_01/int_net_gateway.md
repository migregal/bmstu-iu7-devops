# Table of Contents

1. [Requirements](#requirements)
1. [VirtualBox DHCP](#cirtualBox-dhcp-configuration)
1. [VMs creation](#vms-creation)
1. [VMs preparation](#vms-preparation)
1. [VMs configuration](#vms-configuration)
    1. [Router](#router)
    1. [Server](#server)

# Requirements

* VirtualBox 6.1 or higher;
* Linux OS iso image:
    * tested with: Ubuntu Server 22.04, AlmaLinux 9.1.

# VirtualBox DHCP configuration

On your host system execute:
```console
$ vboxmanage dhcpserver add --netname intnet --ip 10.10.10.1 --netmask 255.255.255.0 --lowerip 10.10.10.2 --upperip 10.10.10.212 --enable
```

VirtualBox will create new DHCP server for `intnet` network, that will be used as internal network for our VMs.

# VMs creation

* Create 2 VMs:
    * Router
        * Netwrok Adaper 1: Bridged Adapter
        * Netwrok Adaper 2: Internal Network (name should match one from DHCP server cfg created before)
    * Server
        * Netwrok Adaper 1: Internal Network (name should match one from router)

* Install desired OS on both VMs (any combination supported)

# VMs preparation:

Any VM configuration will require you to use `root` user privileges.

## Ubuntu Server 22.04
If your're not root yet, run this:
```console
$ sudo su
```

## AlmaLinux 9.1
If your're not root yet, run this:
```console
$ su -m
```

## Router (Ubuntu Server 22.04)

Install required packages:

```console
$ apt-get update && apt-get upgrade -y
$ apt-get install iptables-persistent
```

## Router (AlmaLinux 9.1)

Install required packages:

```console
$ dnf update -y
$ dnf install -y iptables-services
```

# VMs configuration

## Router

Acquire router VM IP to use it later:

```console
$ ip a | grep 'inet'
```

Take one that matches `192.168.*`.

Enable IP forwarding for IPv4 to serve as internal network gateway.

_**Important**_ it's a better to edit `gateway.conf` by hand to prevent conflicts.

```console
$ echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/gateway.conf
$ sysctl -p /etc/sysctl.d/gateway.conf
```

Configure `netfilter` brandmauer to forward traffic.

_**Important**_:
* `<ext>` - place here your interface for Internet interface (i.e. interface from bridged adapter)
* `<int>` - place here your interface for internal network
* To find them, use: `$ ip a`

```console
$ iptables -A FORWARD -i <int> -o <ext> -j ACCEPT

$ iptables -A FORWARD -i <int> -o <ext> -m state --state RELATED,ESTABLISHED -j ACCEPT

$ iptables -t nat -A POSTROUTING -o <ext> -j MASQUERADE
```

### Ubuntu Server 22.04

Save created `iptables` rules to use them even after system reboot:

```console
$ netfilter-persistent save
```

### AlmaLinux 9.1

Save created `iptables` rules to use them even after system reboot:
```console
$ systemctl start iptables
$ systemctl enable iptables
$ service iptables save
```

## Server

_**Important**_
During configuration steps this VM **doesn't have any** Internet access i.e. you can't update any package or install new one, `ping google.com` etc.

Find Internal Network interface by using:

```console
$ ip a
```

Take one that doesn't start match `127.0.0.1` (there should ne the only option because we have just one interface)

### Ubuntu Server 22.04

Find `netplan` default configuration file. To check all files use:

```console
$ ls /etc/netplan/
```

Use your favorite editor (`vi`/`vim`/`nano`) and edit config to be like:

```yaml
netwrok:
  ethernets:
    # ... here maybe smth else
      <here is your internal network interface>:
        dhcp4: true
        routes:
          - to: default
            via: <IPv4 address of router VM>
        nameservers:
          addresses: [8.8.8.8,8.8.4.4]
    # ... here maybe smth else
```

Try new configuration rules:

```console
$ netplan try
```

If `Configuration accpeted` displayed, then you can apply it.

Also you can do it manually:
```console
$ netplan apply
```

Restart network service to apply new rules:
```console
$ systemctl restart system-networkd
```

Verify, that you have Internet access from server VM:

```console
$ ping google.com
```

## AlmaLinux 9.1

_**Important**_ this scenario woll work in all Linux distros where `nmcli` is preinstalled (CentsOS/Red Hat/Fedora/AlmaLinux etc).

In steps below `<int>` is the name of your internal network interface.

```console
$ nmcli con edit <int>
nmcli> goto ipv4
nmcli ipv4> set addresses <IP addresses range from DHCP creation>
nmcli ipv4> set method manual
nmcli ipv4> set gateway <router VM IP>
nmcli ipv4> set dns 8.8.8.8 8.8.4.4
nmcli ipv4> save
$ nmcli con reload <int>
$ nmcli con down <int> && nmcli con up <int>
```

Verify, that you have Internet access from server VM:

```console
$ ping google.com
```
