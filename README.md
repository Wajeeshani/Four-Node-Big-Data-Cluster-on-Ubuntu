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

#Install MariaDB in 3 Master nodes

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

#Installing HAProxy on Ubuntu Slave Node

```
#Install HAProxy
sudo apt install haproxy -y

#Verify installation
haproxy -v
```

#Setting up MariaDB Galera Cluster

A MariaDB Galera Cluster is a synchronous multi-master database cluster solution that provides high availability, scalability, and data consistency across multiple nodes. It allows read/write operations on any node, with changes automatically replicated to all other nodes in real time. 

###Key Features of MariaDB Galera Cluster
- Synchronous Replication: Changes are applied to all nodes simultaneously, ensuring no data loss 16.
- Multi-Master Architecture: Any node can handle write operations, eliminating single-point-of-failure 47.
- High Availability: If a node fails, others continue serving requests 69.
- Automatic Node Recovery: Failed nodes can rejoin the cluster and sync automatically
