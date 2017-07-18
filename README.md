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

Download vim:

```
sudo apt-get install vim
```

Specify the following information in `/etc/wpa_supplicant/wpa_supplicant.conf`

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



### Install Java

Install Java and check the version:

```
sudo apt-get install oracle-java8-jdk
java -version
```



### Ethernet switch

I followed the instructions on how to inter-connect the Raspberry Pi's using an ethernet switch here:
http://makezine.com/projects/build-a-compact-4-node-raspberry-pi-cluster/

The following configuration applies to the master node only.

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



### Create a new group and user

Create a new group and user:

```
sudo addgroup hadoop  
sudo adduser --ingroup hadoop hduser  
sudo adduser hduser sudo
```

and change to this new user:

```
su hduser
```



### SSH configuration

On the master node, generate SSH keys:

```
mkdir ~/.ssh
ssh-keygen -t rsa -P ""
cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys
```

Share the public key from the master node with all slave nodes:

```
ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@192.168.50.11
ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@192.168.50.12
```



### Install Hadoop

I followed the instructions here:
   * http://www.widriksson.com/raspberry-pi-hadoop-cluster/
   * https://blogs.sap.com/2015/04/25/a-hadoop-data-lab-project-on-raspberry-pi-part-14/
   * https://web.archive.org/web/20170221231927/http://www.becausewecangeek.com/building-a-raspberry-pi-hadoop-cluster-part-1/

Download the latest Hadoop binary from http://hadoop.apache.org/releases.html
and test the tarball checksum using SHA-256:

```
wget http://apache.hippo.nl/hadoop/common/hadoop-2.8.0/hadoop-2.8.0.tar.gz
shasum -a 256 hadoop-2.8.0.tar.gz
```

Extract the tarball:

```
sudo tar -xvzf hadoop-2.8.0.tar.gz -C /opt/  
cd /opt
sudo chown -R hduser:hadoop hadoop-2.8.0/  
```

Add the following lines at the end of `~/.bashrc`

```
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")
export HADOOP_INSTALL=/opt/hadoop-2.8.0
export PATH=$PATH:$HADOOP_INSTALL/bin
```

and try running:

```
hadoop version
```



### Configure Hadoop

Configure Hadoop environment variables in `/opt/hadoop-2.8.0/etc/hadoop/hadoop-env.sh` on all nodes:

```
# The java implementation to use. Required.
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")

# The maximum amount of heap to use, in MB. Default is 1000.
export HADOOP_HEAPSIZE=250
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/core-site.xml` on the master node:

```
<configuration>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/hdfs/tmp</value>
  </property>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://hadoop-master:54310</value>
  </property>
</configuration>
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/mapred-site.xml` on the master node:

```
<configuration>
  <property>
    <name>mapred.job.tracker</name>
    <value>hadoop-cluster:54311</value>
  </property>
</configuration>
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/hdfs-site.xml` on the master node:

```
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/yarn-site.xml` on the master node:

````
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>4</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>1024</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>128</value>
  </property>
  <property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>1024</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-vcores</name>
    <value>1</value>
  </property>
  <property>
    <name>yarn.scheduler.maximum-allocation-vcores</name>
    <value>4</value>
  </property>
</configuration>
````

Edit `/opt/hadoop-2.8.0/etc/hadoop/slaves`

```
hadoop-slave1
hadoop-slave2
```



### Create HDFS file system

Create HDFS file system on the master node (name node):
```
sudo mkdir -p /hdfs/tmp
sudo chown hduser:hadoop /hdfs/tmp
sudo chmod 750 /hdfs/tmp
hdfs namenode -format
```



### Start services

Start services:

```
/opt/hadoop-2.8.0/sbin/start-dfs.sh
/opt/hadoop-2.8.0/sbin/start-yarn.sh
```

and check that everything is running:

```
jps
```
