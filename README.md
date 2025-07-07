# Four-Node-Big-Data-Cluster-on-Ubuntu
In this setup, we'll ensure high availability for critical services (HDFS NameNode and YARN ResourceManager) while using all four servers for data storage and processing. To maintain a fault-tolerant quorum for coordination services, we will run them on three of the four available nodes

## Prerequisites (All 4 Servers)
Before begin, ensure each of your 4 servers are ready.<br>
•	Hardware: For the two master/slave nodes, aim for higher RAM (e.g., 32-64GB) to handle both master and slave processes.<br>
• Editor : Install a preferable editor in all 4 nodes<br>
•	Network: All servers must have static IPs and be able to resolve each other's hostnames via the /etc/hosts file.<br>
•	User Account: Create a dedicated hadoop user on all nodes.<br>
•	SSH Access: Configure passwordless SSH for the hadoop user from server-1 to all other servers (server-1, server-2, server-3, server-4).<br>
•	Software: Install OpenJDK 11 and disable or configure firewalls on all four nodes.<br>

## Installing a preferable editor in all 4 nodes
```
## Installing nano
sudo apt update 
provide the password for the sudo login 
sudo apt install nano -y
nano --version
```
## Including statis Ips of all servers to resolve hostname in the /etc/hosts file
```
## Updating host file 
sudo mnano /etc/hosts
```
Include below lines in the hosts file of all 4 nodes
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
cd /opt/haproxy 
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
<value>zkhive:2181</value>
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
12
<property>
<name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
<property>
<name>dfs.namenode.shared.edits.dir</name>
<value>qjournal://zkhive:8485/hadoop-cluster</value>
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
<value>/data/hdfs/journal</value>
</property>
<property>
<name>dfs.datanode.address</name>
<value>0.0.0.0:50010</value>
</property>
<property>
<name>dfs.datanode.http.address</name>
<value>0.0.0.0:50075</value>
</property>
<property>
<name>dfs.hosts.exclude</name>
<value>/opt/hadoop/etc/hadoop/dfs.exclude</value>
</property>
<property>
<name>dfs.namenode.name.dir</name>
<value>/data/hdfs/namenode</value>
</property>
<property>
<name>dfs.namenode.checkpoint.dir</name>
<value>/data/hdfs/checkpoint</value>
</property>
<property>
<name>dfs.datanode.data.dir</name>
<value>/data/hdfs/datanode</value>
</property>
<property>
<name>dfs.replication</name>
<value>1</value>
13
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
<value>/data/hdfs/namenode/edits</value>
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
<value>zkhive:2181</value>
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

```
nano $HADOOP_HOME/etc/hadoop/workers

add
slv1
slv2
slv3
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
 Sudo nano /etc/mysql/mariadb.conf.d/60-galera.cnf
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


