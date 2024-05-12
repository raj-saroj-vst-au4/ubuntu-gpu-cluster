# ubuntu-gpu-cluster

########################################################
### How to Make a Cluster Computer

# Installation of SSH on all the nodes (Login and Compute)
sudo apt update
sudo apt install openssh-server
sudo systemctl status ssh
sudo ufw allow ssh

# Check if SSH is working correctly
ps -A | grep sshd
sudo ss -lnp | grep sshd
ssh -v localhost

# Enable Passwordless Entry on Compute Nodes and Server (Login Node
ssh-keygen -t rsa
ssh-copy-id user@192.168.1.2

# Editing the hosts file
sudo nano /etc/hosts
192.168.1.1 HRG-Server
192.168.7.2 HRG-01

# The Network File System
# NFS on the Login Node
sudo apt-get install nfs-server
sudo mkdir /nfs
sudo nano /etc/exports
/nfs *(rw,sync)
sudo service nfs-kernel-server restart
ls -ld /nfs
sudo chown hashmi /nfs

# NFS on the Client Machines (Compute Nodes)
sudo apt-get install nfs-client
sudo mkdir /nfs
sudo nano /etc/fstab
HRG-Server:/nfs /nfs nfs
sudo systemctl daemon-reload
sudo mount -a

### Installing SLURM ###
# Installing Slurm on the Login Node

$ export MUNGEUSER=1001 
$ sudo groupadd -g $MUNGEUSER munge 
$ sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge 
$ export SLURMUSER=1002 
$ sudo groupadd -g $SLURMUSER slurm 
$ sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
sudo apt-get install -y munge
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo scp /etc/munge/munge.key /nfs/slurm/
sudo systemctl enable munge
sudo systemctl start munge

# Install slurm and associated components on slurm controller (Login) node
sudo apt-get install mariadb-server
sudo apt-get install slurmdbd
sudo apt-get install slurm-wlm

# Create and configure the slurm_acct_db database: (Login Node)
sudo –I (login as root. su command may also be used)
mysql
grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'hashmi12' with grant option; 
create database slurm_acct_db;
exit
sudo mkdir /etc/slurm-llnl
sudo nano /etc/slurm-llnl/slurmdbd.conf (Add the below lines shown in green in the file and save)

AuthType=auth/munge
DbdAddr=localhost
#DbdHost=master0
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
DebugLevel=4
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageLoc=slurm_acct_db
StoragePass=hashmi12
StorageUser=slurm

###Setting database purge parameters
PurgeEventAfter=12months
PurgeJobAfter=12months
PurgeResvAfter=2months
PurgeStepAfter=2months
PurgeSuspendAfter=1month
PurgeTXNAfter=12months
PurgeUsageAfter=12months

# Now we need to give ownership of this file.
chown slurm:slurm /etc/slurm/slurmdbd.conf
chmod -R 600 slurmdbd.conf

# Configuration file /etc/slurm/slurm.conf:
# Visit the website (https://slurm.schedmd.com/configurator.easy.html) to generate a slurm configuration file
sudo nano /etc/slurm-llnl/slurm.conf

# Allow the ports in firewall
sudo ufw allow 6817
sudo ufw allow 6818
sudo ufw allow 6819

# On the master node: (login as root and then run all the below commands)
mkdir /var/spool/slurmctld
chown slurm:slurm /var/spool/slurmctld
chmod 755 /var/spool/slurmctld
mkdir  /var/log/slurm
touch /var/log/slurm/slurmctld.log
touch /var/log/slurm/slurm_jobacct.log /var/log/slurm/slurm_jobcomp.log
chown -R slurm:slurm /var/log/slurm/
chmod 755 /var/log/slurm

#Search and change location of PID file
find / -name "slurmctld.service"
find / -name "slurmd.service"
find / -name "slurmdbd.service"

nano /usr/lib/systemd/system/slurmctld.service
nano /usr/lib/systemd/system/slurmdbd.service
nano /usr/lib/systemd/system/slurmd.service
# Run the following as root
echo CgroupMountpoint=/sys/fs/cgroup >> /etc/slurm-llnl/cgroup.conf

slurmd -C
# Start SLURM Services on Login Node
systemctl daemon-reload
systemctl enable slurmdbd
systemctl start slurmdbd
systemctl enable slurmctld
systemctl start slurmctld
# At this point see the status of the started services:
systemctl status slurmdbd
systemctl status slurmctld
# If any of the services are not active, try rebooting the PC and then check again. Hopefully that will do the job.

### Installing Slurm on the Compute Nodes ###
$ export MUNGEUSER=2001 
$ sudo groupadd -g $MUNGEUSER munge 
$ sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge 
$ export SLURMUSER=2002 
$ sudo groupadd -g $SLURMUSER slurm 
$ sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
sudo apt-get install -y munge

# Now copy the munge authentication key from /nfs/slurm/ on every node.
sudo scp /nfs/slurm/munge.key /etc/munge/
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable munge
sudo systemctl start munge
# Start Installation of SLURM (On all compute nodes)
sudo apt-get install slurm-wlm

# Copy slurm.conf and slurmdbd.conf from the login node to the compute nodes.
# Copy both slurm.conf and slurmdbd.conf to each node at /etc/slurm
sudo scp /nfs/slurm/slurm.conf /etc/slurm
sudo scp /nfs/slurm/slurmdbd.conf /etc/slurm
# On the compute nodes: (login as root and then run all the below commands till 3.5.10)
mkdir /var/spool/slurmd 
chown slurm: /var/spool/slurmd
chmod 755 /var/spool/slurmd
mkdir /var/log/slurm/
touch /var/log/slurm/slurmd.log
chown -R slurm:slurm /var/log/slurm/slurmd.log
chmod 755 /var/log/slurm
mkdir /run/slurm
touch /run/slurm/slurmd.pid (For compute node)
chown slurm /run/slurm
chown slurm:slurm /run/slurm
chmod -R 770 /run/slurm

nano /usr/lib/systemd/system/slurmd.service
echo CgroupMountpoint=/sys/fs/cgroup &gt &gt /etc/slurm/cgroup.conf
slurmd -C
systemctl enable slurmd.service 
systemctl start slurmd.service 
systemctl status slurmd.service
# If the service is active, you are all good, otherwise just reboot the node and reconnect the NFS. Then check the status of slurmd.service. In any case, a reboot at this stage is necessary.
# The following command can check the connectivity with the controller node:
scontrol ping
