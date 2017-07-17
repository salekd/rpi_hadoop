# Raspberry Pi Hadoop Cluster

### SD cards

https://www.raspberrypi.org/downloads/raspbian/
https://www.raspberrypi.org/documentation/installation/installing-images/mac.md
diskutil list
diskutil unmountDisk /dev/disk4
sudo dd bs=1m if=Desktop/2017-07-05-raspbian-jessie-lite.img of=/dev/disk4


### Connect to Wifi

Enable SSH and set hostname in the Raspberry Pi Software Configuration Tool

sudo raspi-config


specify the following in /etc/wpa_supplicant/wpa_supplicant.conf

network={
    ssid=""
    psk=""
}


sudo ifdown wlan0 && sudo ifup wlan0
ifconfig wlan0

check your local network settings:

ip -4 addr show dev wlan0 | grep inet
ip route

check the address of the DNS server:

cat /etc/resolv.conf


### Switch

http://makezine.com/projects/build-a-compact-4-node-raspberry-pi-cluster/

edit /etc/network/interfaces as follows:

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
    
    


sudo apt-get install isc-dhcp-server

edit /etc/dhcp/dhcpd.conf file as follows:

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


edit /etc/hosts as follows:

127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

127.0.1.1	hadoop-master

192.168.50.1	hadoop-master
192.168.50.11	hadoop-slave1
192.168.50.12	hadoop-slave2



check your lacal network settings:

ip -4 addr show dev eth0 | grep inet
ip route

ping 192.168.50.11
