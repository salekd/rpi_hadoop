# Raspberry Pi Hadoop Cluster



### SD cards with Raspbian image

Download the Raspbian image from
https://www.raspberrypi.org/downloads/raspbian/
and follow the instructions on how to install an image on an SD card here:
https://www.raspberrypi.org/documentation/installation/installing-images/mac.md

```
diskutil list
diskutil unmountDisk /dev/disk4
sudo dd bs=1m if=2017-07-05-raspbian-jessie-lite.img of=/dev/disk4
```



### Connect to internet using WiFi

Enable SSH and set hostname in the Raspberry Pi Software Configuration Tool.
I am using the following three hostnames: `hadoop-master`, `hadoop-slave1` and `hadoop-slave2`.

```
sudo raspi-config
```

Specify the following informatino in `/etc/wpa_supplicant/wpa_supplicant.conf`

```
network={
    ssid=""
    psk=""
}
```

Restart WLAN:

```
sudo ifdown wlan0 && sudo ifup wlan0
```

Check your local network settings:

```
ifconfig wlan0
ip -4 addr show dev wlan0 | grep inet
ip route
```

and check the address of the DNS server:

```
cat /etc/resolv.conf
```



### Ethernet switch

I followed the instructions on how to inter-connect the Raspberry Pi's using an ethernet switch here:
http://makezine.com/projects/build-a-compact-4-node-raspberry-pi-cluster/

Edit `/etc/network/interfaces` as follows:

```
auto eth1
allow-hotplug eth1
iface eth1 inet dhcp

auto eth0
allow-hotplug eth0
iface eth0 inet static
    address 192.168.50.1
    netmask 255.255.255.0
    network 192.168.50.0
    broadcast 192.168.50.255
```    
    
Download the following package:

```
sudo apt-get install isc-dhcp-server
```

and edit `/etc/dhcp/dhcpd.conf` as it is shown below (change the MAC addresses accordingly).

```
# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
log-facility local7;

# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.

subnet 192.168.1.0 netmask 255.255.255.0 {
}

group {
   option broadcast-address 192.168.50.255;
   option routers 192.168.50.1;
   default-lease-time 600;
   max-lease-time 7200;
   option domain-name "cluster";
   option domain-name-servers 8.8.8.8, 8.8.4.4;
   subnet 192.168.50.0 netmask 255.255.255.0 {
      range 192.168.50.13 192.168.50.250;

      host hadoop-master {
         hardware ethernet b8:27:eb:10:5c:b5;
         fixed-address 192.168.50.1;
      }
      host hadoop-slave1 {
         hardware ethernet b8:27:eb:45:1e:78;
         fixed-address 192.168.50.11;
      }
      host hadoop-slave2 {
         hardware ethernet b8:27:eb:d4:54:ab;
         fixed-address 192.168.50.12;
      }
   }
}
```

Specify the IP addresses of all three Raspberry Pi's in `/etc/hosts` as follows:

```
127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

127.0.1.1	hadoop-master

192.168.50.1	hadoop-master
192.168.50.11	hadoop-slave1
192.168.50.12	hadoop-slave2
```

You can check your local network settings on all three Raspberry Pi's and ping the internal hosts:

```
ip -4 addr show dev eth0 | grep inet
ip route

ping 192.168.50.11
```
