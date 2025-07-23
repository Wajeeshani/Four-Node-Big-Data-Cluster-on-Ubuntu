# Four-Node High-Availability Hadoop Cluster on Ubuntu

![Hadoop Cluster Architecture](https://img.shields.io/badge/Architecture-High%20Availability-brightgreen)
![Hadoop Version](https://img.shields.io/badge/Hadoop-3.4.0-blue)
![Hive Version](https://img.shields.io/badge/Hive-4.0.0-skyblue)
![MariaDB Version](https://img.shields.io/badge/MariaDB-10.6.12-yellow)
![Spark Version](https://img.shields.io/badge/Spark-3.5.0-orange)

A complete guide to setting up a fault-tolerant, high-availability Hadoop cluster with 3 master nodes and 1 worker node, including HDFS, YARN, ZooKeeper, MariaDB Galera Cluster, Hive, Tez, and Spark integration.

## Cluster Architecture Overview

Our architecture provides high availability for critical services (HDFS NameNode and YARN ResourceManager) while utilizing all four servers for data storage and processing. The quorum-based coordination services run on three nodes for fault tolerance.

### Node Roles:
- **Master Nodes (mst1, mst2, mst3)**:
  - HDFS NameNode (Active/Standby)
  - YARN ResourceManager
  - ZooKeeper
  - JournalNode
  - MariaDB Galera Cluster
  - Hive Metastore
  - Spark Master

- **Worker Node (slv1)**:
  - HDFS DataNode
  - YARN NodeManager
  - Spark Worker
  - HAProxy Load Balancer

## Key Features
- **High Availability HDFS**: Automatic failover for NameNode with ZooKeeper
- **Fault-tolerant YARN**: ResourceManager HA with ZooKeeper-based leader election
- **Synchronous Database**: MariaDB Galera Cluster for Hive Metastore
- **Multi-engine Support**: MapReduce, Tez, and Spark execution engines
- **Load Balancing**: HAProxy for MariaDB read/write splitting
- **Monitoring**: Built-in web UIs for all services

## Prerequisites (All 4 Servers)

### Hardware Requirements
| Node Type       | CPU Cores | RAM  | Storage |
|-----------------|-----------|------|---------|
| Master Nodes    | 8+        | 32GB | 100GB   |
| Worker Node     | 4+        | 16GB | 500GB   |

### Software Requirements
- Ubuntu 20.04/22.04 LTS
- OpenJDK 11
- SSH server
- Python 3

### Network Configuration
All servers must have:
- Static IP addresses
- Proper hostname resolution via `/etc/hosts`
- Passwordless SSH between all nodes
- Open ports for Hadoop services (8020, 8088, 9870, etc.)

## Installation Guide

### 1. System Preparation
```bash
# On all nodes
sudo apt update && sudo apt upgrade -y
sudo apt install -y nano openjdk-11-jdk openssh-server python3
```
## Updating host file -Including static Ips of all servers to resolve hostname in the /etc/hosts file
```
sudo nano /etc/hosts
```
Include below lines in the hosts file of all 4 nodes.
```
# NameNode IP
172.27.16.193    mst1
172.27.16.194    mst2
172.27.16.195    mst3

# DataNode IPs
172.27.16.196    slv1
172.27.16.195    slv2
172.27.16.194    slv3

# JournalNode cluster alias
172.27.16.193    hadoop-cluster
172.27.16.194    hadoop-cluster
172.27.16.195    hadoop-cluster

# mariadb
172.27.16.193    mariadb1
172.27.16.194    mariadb2
172.27.16.195    mariadb3

#Zookeeper alias
172.27.16.193    zkhive
172.27.16.194    zkhive
172.27.16.195    zkhive
```
## Configure passwordless SSH for the hadoop user from server-1 to all other servers

### Prerequisites

- SSH server should be installed and running on all nodes<br>
- You shall have sudo access or root privileges on all nodes

Complete Solution for Mutual SSH Access:
```
#First, ensure all nodes have proper SSH directory structure
sudo su - hadoop
for node in mst1 mst2 mst3 slv1; do
  ssh hadoop@$node "mkdir -p ~/.ssh; chmod 700 ~/.ssh"
done

#Generate SSH keys on all nodes
for node in mst1 mst2 mst3 slv1; do
  ssh hadoop@$node "if [ ! -f ~/.ssh/id_rsa ]; then ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa; fi"
done

#Create combined known_hosts file
# Create empty file
> /tmp/combined_known_hosts
# Collect host keys from all nodes
for node in mst1 mst2 mst3 slv1; do
  ssh-keyscan $node >> /tmp/combined_known_hosts
done
# Verify the content
cat /tmp/combined_known_hosts

#Create combined authorized_keys file
# Create empty file
> /tmp/combined_keys
# Collect public keys from all nodes
for node in mst1 mst2 mst3 slv1; do
  ssh hadoop@$node "cat ~/.ssh/id_rsa.pub" >> /tmp/combined_keys
done
# Verify the content
cat /tmp/combined_keys

#Distribute both files to all node
for node in mst1 mst2 mst3 slv1; do
  echo "Configuring $node..."
  
  # Create .ssh directory if missing
  ssh hadoop@$node "mkdir -p ~/.ssh"
  
  # Copy known_hosts
  cat /tmp/combined_known_hosts | ssh hadoop@$node "cat > ~/.ssh/known_hosts"
  
  # Copy authorized_keys
  cat /tmp/combined_keys | ssh hadoop@$node "cat > ~/.ssh/authorized_keys"
  
  # Set permissions
  ssh hadoop@$node "chmod 700 ~/.ssh; chmod 600 ~/.ssh/*"
done

# Also configure current node
cat /tmp/combined_known_hosts > ~/.ssh/known_hosts
cat /tmp/combined_keys > ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*

#Clean up temporary files
rm /tmp/combined_known_hosts /tmp/combined_keys

#Verify the setup
# you can either use
ssh hadoop@mst2 date # use all nodes to test if date is returned
or
for source in mst1 mst2 mst3 slv1; do
  for target in mst1 mst2 mst3 slv1; do
    echo "Testing $source -> $target..."
    ssh hadoop@$source "ssh hadoop@$target 'echo Success from \$(hostname) to \$(hostname)'"
  done
done
```
verification should output below 

![image](https://github.com/user-attachments/assets/e972f5e8-236d-42bf-a0c4-7907540a682a)

## Software: Install OpenJDK 11 on all four servers

```
#Update the package index
sudo apt update

#Install OpenJDK 11
sudo apt install openjdk-11-jdk -y

#Verify the installation
java -version

#JAVA_HOME setup
#Add the following to your .bashrc or .profile
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

source ~/.bashrc
```
Now the pre-requisits are all installed and configured. 

# Install MariaDB in 3 Master nodes

```
#Install MariaDB
sudo apt install mariadb-server -y

#Start and Enable MariaDB Service
sudo systemctl start mariadb
sudo systemctl enable mariadb

#Verify MariaDB Installation
sudo systemctl status mariadb
```
Verification should output "Active: active (running)"

# Installing HAProxy on Ubuntu Slave Node

```
#Install HAProxy
sudo apt install build-essential libssl-dev zlib1g-dev -y

sudo apt install haproxy -y

#Verify installation
haproxy -v
```

# Installing Zookeeper on Mater nodes

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

```
#Update System Packages
sudo apt update
sudo apt upgrade -y
```
## Zookeeper COnfiguartion

### Create myid Files
On each node, create the myid file matching its assigned number

```
# On zk1:
echo 1 | sudo tee /opt/zookeeper/data/myid

# On zk2:
echo 2 | sudo tee /opt/zookeeper/data/myid

# On zk3:
echo 3 | sudo tee /opt/zookeeper/data/myid
```
### Common zoo.cfg
All nodes should have this identical base configuration in /etc/zookeeper/conf/zoo.cfg

```
sudo nano /opt/zookeeper/conf/conf/zoo.cfg

tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper/data
clientPort=2181
maxClientCnxns=60
autopurge.snapRetainCount=3
autopurge.purgeInterval=1

# Cluster configuration
server.1=mst1:2888:3888
server.2=mst2:2888:3888
server.3=mst3:2888:3888
```
# HADOOP Installation and configuration
```
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
tar xzvf hadoop-3.4.0.tar.gz
sudo mv hadoop-3.4.0 /opt/hadoop
```
Open ~/.bashrc file and add below configurations 

```
export HADOOP_HOME=/opt/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export JAVA_HOME=<path_to_java_jdk>
export PATH=$PATH:$JAVA_HOME/bin

source ~/.bashrc
```
Hadoop Configuration for Multi-node Cluster

You may do following changes on the mst1 and copy the final directory to other nodes 
```
cd /opt/hadoop/etc/hadoop/
nano $HADOOP_HOME/etc/hadoop/hadoop-env.sh
  #Add export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 
```
```
nano $HADOOP_HOME/etc/hadoop/core-site.xml

<configuration>
<property>
<name>fs.defaultFS</name>
<value> hdfs://hadoop-cluster</value>
</property>
<property>
<name>hadoop.tmp.dir</name>
<value>/opt/hadoop-${user.name}</value>
</property>
<property>
<name>ha.zookeeper.quorum</name>
<value>mst1:2181,mst2:2181,mst3:2181</value>
</property>
</configuration>
```

```
nano $HADOOP_HOME/etc/hadoop/hdfs-site.xml

Add following :

<configuration>
<property>
<name>dfs.nameservices</name>
<value>hadoop-cluster</value>
</property>
<property>
<name>dfs.ha.namenodes.hadoop-cluster</name>
<value>nn1,nn2,nn3</value>
</property>
<property>
<name>dfs.namenode.rpc-address.hadoop-cluster.nn1</name>
<value>mst1:8020</value>
</property>
<property>
<name>dfs.namenode.rpc-address.hadoop-cluster.nn2</name>
<value>mst2:8020</value>
</property>
<property>
<name>dfs.namenode.rpc-address.hadoop-cluster.nn3</name>
<value>mst3:8020</value>
</property>
<property>
<name>dfs.namenode.http-address</name>
<value>0.0.0.0:9870</value>
</property>
<property>
<name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
<property>
<name>dfs.namenode.shared.edits.dir</name>
<value>qjournal://mst1:8485;mst2:8485;mst3:8485/hadoop-cluster</value>
</property>
<property>
<name>dfs.ha.fencing.methods</name>
<value>sshfence</value>
</property>
<property>
<name>dfs.ha.fencing.ssh.private-key-files</name>
<value>/home/hadoop/.ssh/id_rsa</value>
</property>
<property>
<name>dfs.ha.automatic-failover.enabled</name>
<value>true</value>
</property>
<property>
<name>dfs.ha.nn.not-become-active-in-safemode</name>
<value>true</value>
</property>
<property>
<name>dfs.journalnode.edits.dir</name>
<value>/opt/hadoop/data/journal</value>
</property>
<property>
<name>dfs.datanode.address</name>
<value>0.0.0.0:50010</value>
</property>
<property>
<name>dfs.datanode.http.address</name>
<value>0.0.0.0:50075</value>
</property>
<!--property>
<name>dfs.hosts.exclude</name>
<value>/opt/hadoop/etc/hadoop/dfs.exclude</value>
</property-->
<property>
<name>dfs.namenode.name.dir</name>
<value>/opt/hadoop/data/namenode</value>
</property>
<property>
<name>dfs.namenode.checkpoint.dir</name>
<value>/opt/hadoop/data/checkpoint</value>
</property>
<property>
<name>dfs.datanode.data.dir</name>
<value>/opt/hadoop/data/datanode</value>
</property>
<property>
<name>dfs.replication</name>
<value>2</value>
</property>
<property>
<name>dfs.blocksize</name>
<value>134217728</value> <!-- 128 MB -->
</property>
<property>
<name>dfs.heartbeat.interval</name>
<value>3</value>
</property>
<property>
<name>dfs.namenode.checkpoint.period</name>
<value>3600</value> <!-- Every hour -->
</property>
<property>
<name>dfs.permissions.enabled</name>
<value>true</value>
</property>
<property>
<name>dfs.namenode.edits.journal-plugin.hdfs</name>
<value>org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager</value>
</property>
<property>
<name>dfs.namenode.edits.dir</name>
<value>/opt/hadoop/data/namenode/edits</value>
</property>
<property>
<name>dfs.datanode.ipc.address</name>
<value>0.0.0.0:8010</value>
</property>
<property>
<name>dfs.user.home.dir.prefix</name>
<value>/user</value>
</property>
<property>
<name>dfs.namenode.acls.enabled</name>
<value>true</value>
</property>
</configuration>
```
```
nano $HADOOP_HOME/etc/hadoop/yarn-site.xml

<configuration>

<!-- Site specific YARN configuration properties -->


<property>
<name>yarn.resourcemanager.ha.enabled</name>
<value>true</value>
</property>
<property>
<name>yarn.resourcemanager.cluster-id</name>
<value>hadoop-cluster</value>
</property>
<property>
<name>yarn.resourcemanager.ha.rm-ids</name>
<value>rm1,rm2,rm3</value>
</property>
<property>
<name>yarn.resourcemanager.hostname.rm1</name>
<value>mst1</value>
</property>
<property>
<name>yarn.resourcemanager.hostname.rm2</name>
<value>mst2</value>
</property>
<property>
<name>yarn.resourcemanager.hostname.rm3</name>
<value>mst3</value>
</property>
<property>
<name>yarn.resourcemanager.address.rm1</name>
<value>mst1:8032</value>
</property>
<property>
<name>yarn.resourcemanager.address.rm2</name>
<value>mst2:8032</value>
</property>
<property>
<name>yarn.resourcemanager.address.rm3</name>
<value>mst3:8032</value>
</property>
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
<property>
<name>yarn.resourcemanager.webapp.address.rm1</name>
<value>mst1:8088</value>
</property>
<property>
<name>yarn.resourcemanager.webapp.address.rm2</name>
<value>mst2:8088</value>
</property>
<property>
<name>yarn.resourcemanager.webapp.address.rm3</name>
<value>mst3:8088</value>
</property>
<property>
<name>yarn.log-aggregation-enable</name>
<value>true</value>
</property>
<property>
<name>yarn.nodemanager.resource.memory-mb</name>
<value>6144</value>
</property>
<property>
<name>yarn.scheduler.minimum-allocation-mb</name>
<value>1024</value>
</property>
<property>
<name>yarn.scheduler.maximum-allocation-mb</name>
<value>6144</value>
</property>
<property>
<name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
<value>true</value>
</property>
<property>
<name>yarn.client.failover-proxy-provider</name>
<value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
</property>
<property>
<name>yarn.resourcemanager.zk-address</name>
<value>mst1:2181,mst2:2181,mst3:2181</value>
</property>
</configuration>

```
```
nano $HADOOP_HOME/etc/hadoop/mapred-site.xml

<configuration>


<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>
<property>
<name>mapreduce.jobtracker.address</name>
<value>localhost:8032</value>
</property>
<property>
<name>mapreduce.jobhistory.done-dir</name>
<value>/logs/hadoop/jobhistory/done</value>
</property>
<property>
<name>mapreduce.shuffle.port</name>
<value>13562</value>
</property>
<property>
<name>mapreduce.map.memory.mb</name>
<value>1024</value> <!-- 1 GB -->
</property>
<property>
<name>mapreduce.reduce.memory.mb</name>
<value>1024</value> <!-- 1 GB -->
</property>
<property>
<name>mapreduce.map.java.opts</name>
<value>-Xmx1536m</value>
</property>
<property>
<name>mapreduce.reduce.java.opts</name>
<value>-Xmx3072m</value>
</property>
<property>
<name>mapreduce.jobhistory.intermediate-done-dir</name>
<value>/logs/hadoop/jobhistory/intermediate-done</value>
</property>
<property>
<name>mapreduce.jobhistory.address</name>
<value>0.0.0.0:10020</value>
</property>
<property>
<name>mapreduce.jobhistory.webapp.address</name>
<value>0.0.0.0:19888</value>
</property>
<property>
<name>yarn.app.mapreduce.am.env</name>
<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
<property>
<name>mapreduce.map.env</name>
<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
<property>
<name>mapreduce.reduce.env</name>
<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
</configuration>
```
Make relevent directories in the /opt/hadoop location 
```
cd /opt/hadoop/
mkdir data
cd data
mkdir -p {datanode,namenode}
mkdir checkpoint  
mkdir journal  
```
```
nano $HADOOP_HOME/etc/hadoop/workers

add
slv1
slv2
slv3
```
```
hadoop-daemon.sh start journalnode
#one master
hdfs namenode -format

#Other masters
hdfs --daemon start namenode
hdfs namenode -bootstrapStandby

#HDFS start
start-all.sh
```

## Verification
 
### Check HDFS Status
```bash
hdfs dfsadmin -report
hdfs haadmin -getServiceState nn1
hdfs haadmin -getServiceState nn2
hdfs haadmin -getServiceState nn3
```
### Check YARN Status
```bash
yarn node -list
yarn application -list
```
```
jps
```
you will see the following output 

45889 ResourceManager
33297 QuorumPeerMain
47475 Jps
45259 JournalNode
45004 NameNode
45501 DFSZKFailoverController

# Hive Configuration

## Adding System Variables

You may add below configurations in bashrc file. 
```
nano ~/.bashrc 

#Hive Related Options
export HIVE_HOME=/opt/hive
export HIVE_CONF=$HIVE_HOME/conf
export PATH=$PATH:$HIVE_HOME/bin
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib/*:.
export CLASSPATH=$CLASSPATH:$HIVE_HOME/lib/*:.

source ~/.bashrc 
```


# Setting up hybrid MariaDB Galera Cluster setup with dedicated master and slave nodes

A MariaDB Galera Cluster is a synchronous multi-master database cluster solution that provides high availability, scalability, and data consistency across multiple nodes. It allows read/write operations on any node, with changes automatically replicated to all other nodes in real time. 

### Key Features of MariaDB Galera Cluster
- Synchronous Replication: Changes are applied to all nodes simultaneously, ensuring no data loss
- Multi-Master Architecture: Any node can handle write operations, eliminating single-point-of-failure
- High Availability: If a node fails, others continue serving requests
- Automatic Node Recovery: Failed nodes can rejoin the cluster and sync automatically

### Galera Cluster Setup (Multi-Master: mst1, mst2, mst3)
```
 sudo nano /etc/mysql/mariadb.conf.d/60-galera.cnf
```
Edit the configuration file and include below configurations 
```
[galera]
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "galera_cluster"
wsrep_cluster_address = "gcomm://mst1,mst2,mst3"
wsrep_node_address = "172.27.16.193" # change as per the current node 
wsrep_node_name = "mst1"  # change as per the current node            
binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
log_slave_updates = ON

[mysqld]
bind-address=0.0.0.0
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2

```
```
sudo systemctl set-environment MYSQLD_OPTS="--wsrep-new-cluster"
sudo systemctl start mariadb
sudo systemctl unset-environment MYSQLD_OPTS # IMPORTANT: Unset this immediately after the first node starts
```
# create some users 

```
sudo mysql -u root (password not required)

CREATE USER 'root'@'%' IDENTIFIED BY 'root';
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#now log again with password

```
mysql -u root -p (with password)
CREATE USER 'haproxy_check'@'%';
FLUSH PRIVILEGES;

CREATE USER 'appuser'@'%' IDENTIFIED BY 'appuser';
GRANT ALL PRIVILEGES ON *.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```
# HA Proxy Configurations

Add below configurations in 

```
# Global Configurations 

listen stats
        mode http
        bind *:8404
        stats enable
        stats uri /haproxy?stats
        stats refresh 10s
#       stats realm Strictly\ Private
        stats show-legends
        stats show-node
        stats show-desc "Open Source Hadoop Cluster Statistics"
        stats show-modules
        stats hide-version
        stats auth hadoop:hadoop

# MariaDB Configurations 

listen mariadb_write
    bind *:3307
    mode tcp
    option mysql-check user haproxy_check
    log global
    option tcplog
    server mariadb1 mst1:3306 check
    server mariadb2 mst2:3306 check backup
    server mariadb3 mst3:3306 check backup

# Read requests (load-balanced)
listen mariadb_read
    bind *:3306
    mode tcp
    option mysql-check user haproxy_check
    log global
    option tcplog
    balance roundrobin
    server mariadb1 mst1:3306 check
    server mariadb2 mst2:3306 check
    server mariadb3 mst3:3306 check

frontend mariadb_front
        bind *:3307
        timeout client 30s
        mode tcp
        log global
        option tcplog
        default_backend mariadb_back

backend mariadb_back
        mode tcp
        balance roundrobin
        option tcp-check
        log global
        option tcplog
        server mariadb1 mst1:3306 check
        server mariadb2 mst2:3306 check backup
        server mariadb3 mst3:3306 check backup
        timeout connect 5s
        timeout server 30s
		
```
# Hive Metastore

You need to stop mysql on all nodes and run below commands 

```
#Hive build mqsql metastore
cd $HIVE_HOME/bin
schematool -dbType mysql -initSchema
```
It will run a script and provide you a successfull output
Then you need to run below command to up the metastore

```
hive --service metastore &
hive --service hiveserver2 &
```

Then you can connect to the Hive using below command 

```
hive
beeline -u "jdbc:hive2://mst1:2181,mst2:2181,mst3:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"
```
Then you may check the connectivity by creating a database. 

# TEZ Configurations

Download and move the tez files to /opt/ path

```
wget https://downloads.apache.org/tez/0.10.4/apache-tez-0.10.4-bin.tar.gz
tar -xzvf apache-tez-0.10.4-bin.tar.gz 
mv /opt/apache-tez-0.10.4-bin /opt/tez
```

Add below System variables

```
nano ~/.bashrc 


#Tez Related Options
export TEZ_HOME=/opt/tez
export TEZ_CONF_DIR=$TEZ_HOME/conf

export TEZ_JARS=$TEZ_HOME
# For enabling hive to use the Tez engine
if [ -z "$HIVE_AUX_JARS_PATH" ]; then
export HIVE_AUX_JARS_PATH="$TEZ_JARS"
else
export HIVE_AUX_JARS_PATH="$HIVE_AUX_JARS_PATH:$TEZ_JARS"
fi


export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*

source ~/.bashrc
```

### Create required HDFS folders and upload TEZ
```
hdfs dfs -mkdir /apps
hdfs dfs -mkdir /apps/tez
hdfs dfs -chmod g+w /apps
hdfs dfs -chmod g+wx /apps

cp $HIVE_HOME/lib/protobuf-java-3.24.4.jar $TEZ_HOME/lib/
hdfs dfs -put $HIVE_HOME/lib/hive-exec-4.0.0.jar /apps/tez
cd $TEZ_HOME 
hdfs dfs -put * /apps/tez
```
### TEZ Configurations (tez-site.xml)

```
cd $TEZ_HOME/conf/

nano tez-site.xml 
```
add below configurations
```
<?xml version="1.0"?>
<configuration>

  <property>
    <name>tez.lib.uris</name>
    <value>${fs.default.name}/apps/tez/share/tez.tar.gz</value>
  </property>

</configuration>
```
### Hive Configurations - Change execution engine (hive-site.xml)

Add below configurations
```
nano $HIVE_HOME/conf/hive-site.xml 
```
```
  <property>
    <name>hive.execution.engine</name>
    <value>tez</value>
    <description>
      Expects one of [mr, tez, spark].
      Chooses execution engine. Options are: mr (Map reduce, default), tez, spark. While>
      remains the default engine for historical reasons, it is itself a historical engine
      and is deprecated in Hive 2 line. It may be removed without further warning.
    </description>
  </property>
```
once all above configuratins are donw you may copy the folder to other master nodes 

```
 scp -r /opt/tez hadoop@mst2:/opt/
 scp -r /opt/tez hadoop@mst3:/opt/
```
Then you can check if MapReduce has changed to Tez by running below command in the beeline 

```
beeline -u "jdbc:hive2://mst1:2181,mst2:2181,mst3:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"
```
```
set hive.execution.engine;
```
It will output below. 

<img width="656" height="127" alt="image" src="https://github.com/user-attachments/assets/c33465c4-5d2c-4f99-b72f-82c19f64f3fc" />

# SPARK Configurations

Download spark using below code and move the folder to /opt/ path

```
wget https://downloads.apache.org/spark/spark-3.5.0/spark-3.5.0-bin-hadoop3.tgz
tar -xvzf spark-3.5.0-bin-hadoop3.tgz
mv  mv spark-3.5.6-bin-hadoop3 /opt/spark
```

Add below system variables 

```
nano ~/.bashrc 

#Spark Related Options
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
export PYSPARK_PYTHON=/usr/bin/python3
export LD_LIBRARY_PATH=$HADOOP_HOME/lib/native:$LD_LIBRARY_PATH

source ~/.bashrc 
```

```
cp log4j2.properties.template log4j2.properties
nano $SPARK_HOME/conf/log4j2.properties
```
Add below configurations 

```
logger.HiveConf.name=org.apache.hadoop.hive.conf.HiveConf
logger.HiveConf.level=ERROR
```
Copy mysql connector to spark jar folder
```
sudo cp /home/hadoop/mysql-connector-j-9.3.0/mysql-connector-j-9.3.0.jar  $SPARK_HOME/jars/
```
Update spark-env.sh file 
```
cd $SPARK_HOME/conf
cp spark-env.sh.template spark-env.sh
```
Add below configurations in the .sh file

```
export SPARK_HOME=/opt/spark
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export SPARK_MASTER_HOST=172.27.16.193
export SPARK_WORKER_CORES=4
export SPARK_WORKER_MEMORY=4g
export SPARK_WORKER_INSTANCES=1
export SPARK_LOCAL_DIRS=/data/spark
export SPARK_WORKER_DIR=/data/spark/work
export SPARK_MASTER_PORT=7077
export SPARK_MASTER_WEBUI_PORT=8081
export SPARK_LOG_DIR="$SPARK_HOME/logs"
export SPARK_DRIVER_MEMORY=2g
export SPARK_EXECUTOR_MEMORY=4g
export SPARK_HISTORY_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.17.2.jar=19070:/opt/jmx_exporter/exporter.yml"
export SPARK_MASTER_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=mst1:2181,mst2:2181,mst3:2181 -Dspark.deploy.zookeeper.dir=/spark"
```

Copy the spark folder to other nodes
```
scp -r /opt/spark hadoop@mst2:/opt/
scp -r /opt/spark hadoop@mst3:/opt/
scp -r /opt/spark hadoop@slv1:/opt/
```
In the Worker node chnage below congiguration in the spark-env.sh file

```
export SPARK_MASTER=spark://mst1:7077,mst2:7077,mst3:7077
# Add below lines
export SPARK_WORKER_DIR=/opt/spark/work
export SPARK_WORKER_PORT=44821
```

```
#Testing
start-master.sh
start-worker.sh spark://localhost:7077

# start slave node
$SPARK_HOME/sbin/start-worker.sh

netstat -tuln | grep 7077
netstat -tuln | grep 8080
```
<img width="1030" height="177" alt="image" src="https://github.com/user-attachments/assets/45da9c31-6ef4-489e-8311-bb748e5d0ca8" />

Through below URL spark GUI should be available 
http://172.27.16.193:8081/
http://172.27.16.194:8081/
http://172.27.16.195:8081/

# PrestoDB Installation

```
wget https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.292/presto-server-0.292.tar.gz
tar -xzvf presto-server-0.292.tar.gz
mv presto-server-0.292 /opt/prestodb
```

