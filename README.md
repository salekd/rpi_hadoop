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
      host hadoop-slave-1 {
         hardware ethernet b8:27:eb:45:1e:78;
         fixed-address 192.168.50.11;
      }
      host hadoop-slave-2 {
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

192.168.50.1	hadoop-master
192.168.50.11	hadoop-slave-1
192.168.50.12	hadoop-slave-2
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

I followed the instructions from http://www.widriksson.com/raspberry-pi-hadoop-cluster

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

I followed the instructions from https://developer.ibm.com/recipes/tutorials/building-a-hadoop-cluster-with-raspberry-pi/ and https://dqydj.com/raspberry-pi-hadoop-cluster-apache-spark-yarn/

The following files should be modified on the master node. The configuration will be shared with the other nodes afterwards using rsync.

Configure Hadoop environment variables in `/opt/hadoop-2.8.0/etc/hadoop/hadoop-env.sh`:

```
# The java implementation to use. Required.
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")

# The maximum amount of heap to use, in MB. Default is 1000.
export HADOOP_HEAPSIZE=250
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/core-site.xml`:

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

Edit `/opt/hadoop-2.8.0/etc/hadoop/mapred-site.xml`:

```
<configuration>
  <property>
    <name>mapred.job.tracker</name>
    <value>hadoop-master:54311</value>
  </property>
  <property>
    <name>mapred.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/hdfs-site.xml`:

```
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/hdfs/tmp/datanode</value>
    <final>true</final>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/hdfs/tmp/namenode</value>
    <final>true</final>
  </property>
  <property>
    <name>dfs.namenode.http-address</name>
    <value>hadoop-master:50070</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
</configuration>
```

Edit `/opt/hadoop-2.8.0/etc/hadoop/yarn-site.xml`:

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
  
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>hadoop-master:8025</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>hadoop-master:8035</value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>hadoop-master:8050</value>
  </property>
</configuration>
````

Edit `/opt/hadoop-2.8.0/etc/hadoop/slaves`:

```
hadoop-slave1
hadoop-slave2
```

Share the configuration with the other nodes. First install rsync on all nodes.

```
sudo apt-get -y install rsync
```

Synchronize the hadoop directory from the master node:

```
rsync -avxP /opt/hadoop-2.8.0/ hduser@192.168.50.11:/opt/hadoop-2.8.0/
rsync -avxP /opt/hadoop-2.8.0/ hduser@192.168.50.12:/opt/hadoop-2.8.0/
```



### Create HDFS file system

Create HDFS file system on all nodes:
```
sudo mkdir -p /hdfs/tmp
sudo chown hduser:hadoop /hdfs/tmp
sudo chmod 750 /hdfs/tmp
hdfs namenode -format
```



### Start services

Start services on the master node:

```
/opt/hadoop-2.8.0/sbin/start-dfs.sh
/opt/hadoop-2.8.0/sbin/start-yarn.sh
```

and check that everything is running:

```
jps
hdfs dfsadmin -report
```

The available space should correspond to both data nodes in total:

```
hadoop fs -df -h
```

Browse the name node web interface:

```
sudo apt-get install lynx
lynx http://192.168.50.1:8088/
```



### Run a test

Create a test directory in HDFS and upload a file there:
```
hadoop fs -mkdir hdfs://master:54310/testdir/
hadoop fs -df -h
hadoop fs -mkdir hdfs://hadoop-master:54310/testdir/
hadoop fs -copyFromLocal /opt/hadoop-2.8.0/LICENSE.txt hdfs://hadoop-master:54310/testdir/license.txt
hadoop fs -ls hdfs://hadoop-master:54310/testdir/
```

Check the status of the test directory:

```
hdfs fsck /testdir/ -files -blocks -racks
```

The following example is taken from
https://hadoop.apache.org/docs/r2.8.0/hadoop-project-dist/hadoop-common/SingleCluster.html

```
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/hduser
hdfs dfs -put /opt/hadoop-2.8.0/etc/hadoop input
hadoop jar /opt/hadoop-2.8.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.0.jar grep input output 'dfs[a-z.]+'
hdfs dfs -cat output/*
```



### Install and configure Spark

I followed the instructions from https://sites.google.com/a/complexsys.info/scattershot/home/configuring-apache-spark-on-raspbery-pi-2-3-clusters

Download Spark from https://spark.apache.org/downloads.html

```
wget https://d3kbcqa49mib13.cloudfront.net/spark-2.2.0-bin-hadoop2.7.tgz
sudo tar -xvzf spark-2.2.0-bin-hadoop2.7.tgz -C /opt/
sudo chown -R hduser:hadoop /opt/spark-2.2.0-bin-hadoop2.7
```

Create a configuration file `/opt/spark-2.2.0-bin-hadoop2.7/conf/spark-env.sh` with the following settings:

```
SPARK_MASTER_IP=192.168.50.1
SPARK_EXECUTOR_MEMORY=512m
SPARK_DRIVER_MEMORY=512m
SPARK_WORKER_MEMORY=512m
SPARK_DAEMON_MEMORY=512m
```

and make it executable.

```
chmod 755 /opt/spark-2.2.0-bin-hadoop2.7/conf/spark-env.sh
```

Specify the slaves in `/opt/spark-2.2.0-bin-hadoop2.7/conf/slaves`:

```
hadoop-slave1
hadoop-slave2
```

From the slaves, use scp to copy the Spark directory from the master node and change the ownership:

```
sudo scp -r hduser@192.168.50.1:/opt/spark-2.2.0-bin-hadoop2.7 /opt/.
sudo chown -R hduser:hadoop /opt/spark-2.2.0-bin-hadoop2.7
```

Start Spark:

```
/opt/spark-2.2.0-bin-hadoop2.7/sbin/start-all.sh
```

Open the Spark UI:

```
lynx http://192.168.50.1:8080/
```



### Links

Other useful links:
   * https://hadoop.apache.org/docs/r2.8.0/hadoop-project-dist/hadoop-common/ClusterSetup.html
   * http://www.cs.brandeis.edu//~cs147a/lab/hadoop-troubleshooting/
   * https://blogs.sap.com/2015/04/25/a-hadoop-data-lab-project-on-raspberry-pi-part-14/
   * https://web.archive.org/web/20170221231927/http://www.becausewecangeek.com/building-a-raspberry-pi-hadoop-cluster-part-1/
   * https://medium.com/@jasonicarter/how-to-hadoop-at-home-with-raspberry-pi-part-1-3b71f1b8ac4e
   * http://www.nigelpond.com/uploads/How-to-build-a-7-node-Raspberry-Pi-Hadoop-Cluster.pdf
