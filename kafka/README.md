# Deploying a production-ready Kafka cluster with Ansible

# Table Of Contents
1. [Document objective](#1-document-objective)
2. [Hardware requirements](#2-hardware-requirements)
3. [Software requirements](#3-software-requirements)
4. [Kafka cluster architecture](#4-kafka-cluster-architecture)
5. [Deploy the Kafka cluster in local VMWare Workstation machines](#5-deploy-the-kafka-cluster-in-local-vmware-workstation-machines)

5.1. [Enable SSHing into this machine installing a DHCP server](#5-1-enable-sshing-into-this-machine-installing-a-dhcp-server)

5.2. [Install Ansible into this node](#5-2-install-ansible-into-this-node)

5.3 [Clone the Ubuntu machine](#5-3-clone-the-ubuntu-machine)

5.4 [Make their IPs static and add them to /etc/hosts](#5-4-make-their-ips-static-and-add-them-to-/etc/hosts)

5.5 [Deployment Example](#5-5-deployment-example)

5.6 [Verifying the Deployment](#5-6-verifying-the-deployment)

6. [Testing fault tolerance](#6-testing-fault-tolerance)


## 1 Document objective
---------------------------------------------

The present document is a guide to build a three node Kafka cluster machine,
build on top of three Ubuntu 16.04 VMWare virtual machines.

We will use then this environment to run performance and fail tolerance tests,
producing XML message workloads into a Kafka topic, and replicating these messages
to a topic in a Google Cloud Kafka cluster.


## 2 Hardware requirements
------------------------------------------------------------------------------

Three VMWare virtual machines whith this settings per broker:

    32GB of RAM
    X GB HDD Disk, where

    X = (message_size * messages_per_sec * seconds_messages_available * replication_factor) / num_brokers)

Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)
https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines


## 3 Software requirements
------------------------------------------------------------------------------

- Requires Ansible 1.2
- Expects Ubuntu 16.04 hosts


## 4 Kafka cluster architecture
------------------------------------

Three node cluster with Zookeeper and Kafka clusters installed and configured:

![Alt text](images/architecture.png "Architecture")


## 5 Deploy the Kafka cluster in local VMWare Workstation machines
-----------------------------------------------------------------------

Create one Ubuntu virtual machine:

![Alt text](images/vmware1.png "vmware1")

i.e.

username: arturo
password: arturo

![Alt text](images/vmware2.png "vmware2")


### 5.1 Enable SSHing into this machine installing a DHCP server with this bash command:
-------------------------------------------------------------------------------------

```bash
$ sudo apt-get install openssh-server
```
![Alt text](images/enablessh1.png "enablessh1")

Now we will be able to ssh into this machine from a machine in the same domain:

![Alt text](images/enablessh2.png "enablessh2")


### 5.2 Install Ansible into this node:
-----------------------------------------------------------------------

```bash
$ sudo apt-get install ansible
```


### 5.3 Clone the Ubuntu machine
-----------------------------------------------------------------------

![Alt text](images/vmware3.png "vmware3")

Login into the three virtual machines. Name them broker1, broker2 and broker3 in the /etc/hostname file:

```bash
arturo@broker1:~$ sudo vi /etc/hostname
broker1

arturo@broker2:~$ sudo vi /etc/hostname
broker2

arturo@broker3:~$ sudo vi /etc/hostname
broker3

```


### 5.4 Make their IPs static and add them to /etc/hosts
-----------------------------------------------------------------------

make their IPs static:

```bash
arturo@broker1:~$ sudo vi /etc/network/interfaces
auto lo
iface lo inet loopback
auto ens33
iface ens33 inet static
address 192.168.0.116
netmask 255.255.255.0
gateway 192.168.0.1
dns-nameservers 8.8.8.8 192.168.0.1

arturo@broker1:~$ sudo reboot now

arturo@broker2:~$ sudo vi /etc/network/interfaces
auto lo
iface lo inet loopback
auto ens33
iface ens33 inet static
address 192.168.0.117
netmask 255.255.255.0
gateway 192.168.0.1
dns-nameservers 8.8.8.8 192.168.0.1

arturo@broker2:~$ sudo reboot now

arturo@broker3:~$ sudo vi /etc/network/interfaces
auto lo
iface lo inet loopback
auto ens33
iface ens33 inet static
address 192.168.0.118
netmask 255.255.255.0
gateway 192.168.0.1
dns-nameservers 8.8.8.8 192.168.0.1

arturo@broker3:~$ sudo reboot now
```

Add their references into the /etc/hosts:

```bash
arturo@broker1:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
192.168.0.116   broker1
192.168.0.117   broker2
192.168.0.118   broker3

arturo@broker2:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
192.168.0.116   broker1
192.168.0.117   broker2
192.168.0.118   broker3

arturo@broker3:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
192.168.0.116   broker1
192.168.0.117   broker2
192.168.0.118   broker3

```


### 5.5 Deployment Example

Our inventory may look as follows:

```bash
[broker1]
192.168.0.116 ansible_ssh_user=arturo
[broker2]
192.168.0.117 ansible_ssh_user=arturo
[broker3]
192.168.0.118 ansible_ssh_user=arturo
```

We can build the site with the following command:

```bash
$ ansible-playbook kafka.yml --ask-sudo-pass -k --become --become-method=sudo
```



### 5.6 Verifying the Deployment
---------------------------------------------

Once configuration and deployment has completed we can check our local kafka cluster
availability by connecting to any individual node of the Kafka cluster and create
a topic from one node, check the topic from other, produce some messages from one
node into the topic and consume them from the rest of the nodes, i.e.:

```bash
arturo@broker1:/opt/kafka$ bin/kafka-topics.sh --create --zookeeper broker1:2181,broker2:2181,broker3:2181 --replication-factor 3 --partitions 3 --topic test
Created topic "test".

arturo@broker2:/opt/kafka$ bin/kafka-topics.sh --describe --zookeeper broker1:2181,broker2:2181,broker3:2181 --topic test
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 3	Replicas: 3,1,2	Isr: 2,3
	Topic: test	Partition: 1	Leader: 2	Replicas: 1,2,3	Isr: 2,3
	Topic: test	Partition: 2	Leader: 2	Replicas: 2,3,1	Isr: 2,3

arturo@broker3:/opt/kafka$ bin/kafka-topics.sh --list --zookeeper broker1:2181,broker2:2181,broker3:2181
test


arturo@broker1:/opt/kafka$ bin/kafka-console-producer.sh --broker-list broker1:9092,broker2:9092,broker3:9092 --topic test
>mensaje1
mensaje2
mensaje3

arturo@broker2:/opt/kafka$ bin/kafka-console-consumer.sh --bootstrap-server broker1:9092,broker2:9092,broker3:9092 --topic test –from-beginning
mensaje1
mensaje2
mensaje3

arturo@broker3:/opt/kafka$ bin/kafka-console-consumer.sh --bootstrap-server broker1:9092,broker2:9092,broker3:9092 --topic test –from-beginning
mensaje1
mensaje2
mensaje3
```

## 6 Testing fault tolerance
---------------------------------------------

We have deployed Monit in all the brokers to keep alive the Zookeeper and Kafka processes,
so that they are automatically rebooted in a crash eventuality:

```bash
arturo@broker2:~$ cat /etc/monit/conf-available/kafka
check process kafka matching "java -Xmx1G.*kafka"
        start program = "/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties &"
        stop program = "/opt/kafka/bin/kafka-server-stop.sh"
        alert atarin.mistralbs@goldcar.com with reminder on 500 cycles

arturo@broker2:~$ cat /etc/monit/conf-available/zoo
check process zoo
        matching "java -Dzookeeper.log.dir="
        start program = "/opt/zookeeper/bin/zkServer.sh start"
        stop program = "/opt/zookeeper/bin/zkServer.sh stop"
        alert atarin.mistralbs@goldcar.com with reminder on 500 cycles

arturo@broker2:~$ sudo monit -v
[sudo] password for arturo:
/etc/monit/monitrc:289: Include failed -- Success '/etc/monit/conf.d/*'
/etc/monit/monitrc:290: Include failed -- Success '/etc/monit/conf-enabled/*'
Runtime constants:
 Control file       = /etc/monit/monitrc
 Log file           = /var/log/monit.log
 Pid file           = /run/monit.pid
 Id file            = /var/lib/monit/id
 State file         = /var/lib/monit/state
 Debug              = True
 Log                = True
 Use syslog         = False
 Is Daemon          = True
 Use process engine = True
 Limits             = {
                    =   programOutput:     512 B
                    =   sendExpectBuffer:  256 B
                    =   fileContentBuffer: 512 B
                    =   httpContentBuffer: 1024 kB
                    =   networkTimeout:    5 s
                    = }
 Poll time          = 120 seconds with start delay 0 seconds
 Event queue        = base directory /var/lib/monit/events with 100 slots
 Mail from          = (not defined)
 Mail subject       = (not defined)
 Mail message       = (not defined)
 Start monit httpd  = False

The service list contains the following entries:

System Name           = broker2
 Monitoring mode      = active

-------------------------------------------------------------------------------
Monit daemon with PID 1364 awakened

arturo@broker2:~$ cat /var/log/monit.log
[PDT Apr 29 21:56:28] info     :  New Monit id: 0b142149ebc37e56dcd862fddff7b6a7
 Stored in '/var/lib/monit/id'
[PDT Apr 29 21:56:28] info     : Starting Monit 5.16 daemon
[PDT Apr 29 21:56:28] info     : 'broker2' Monit 5.16 started

```

I.e., if we force a crash in the Zookeeper process in broker2, Monit will automatically spin-off a new process:

```bash
arturo@broker2:~$ ps -ef | grep "java -Dzookeeper.log.dir"
root       1306      1  1 22:28 ?        00:00:06 java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE


arturo@broker2:~$ sudo kill -9 1306


arturo@broker2:~$ ps -ef | grep "java -Dzookeeper.log.dir"
root      23777      1 30 22:35 ?        00:00:02 java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE

```

