# HPC Mini-Cluster Installation Guide

## 1. Introduction

A High Performance Computing (HPC) cluster connects multiple computers to execute computational workloads using shared resources. This project covers the configuration of cluster nodes, shared storage, job scheduling, parallel software, and centralized user management.

### Technologies

- Rocky Linux
- VirtualBox
- SLURM
- Munge
- NFS
- OpenLDAP
- OpenMPI
- Spack

## 2. Cluster Architecture

The lab environment described in the installation reference consists of a master node, a login node, and three compute nodes.

| Node | Responsibility |
|---|---|
| Master | Cluster administration, shared storage, scheduler controller, LDAP server |
| Login | User access, compilation, and job submission |
| Compute nodes | Execution of scheduled workloads |

The nodes communicate through an internal network. A NAT adapter provides internet access for package installation.

## 3. Prepare the Virtual Machines

### 3.1 Install VirtualBox

Install VirtualBox on the host operating system and ensure its kernel modules are available.

### 3.2 Download Rocky Linux

Download the Rocky Linux x86_64 Minimal ISO from the official Rocky Linux website.

### 3.3 Create the master virtual machine

Create a Linux virtual machine with the following resources:

- 2 CPU cores
- Approximately 2 GB RAM
- 30 GB virtual disk
- NAT network adapter
- Internal network adapter

Use `hpcnet` as the internal network name for the nodes in this lab.

### 3.4 Install Rocky Linux

Install Rocky Linux using the Minimal Install option. Create an administrative account and configure the hostname of the master node.

## 4. Configure Cluster Networking

Each node requires a unique hostname and static internal IP address.

Configure the internal network interface through NetworkManager:

```bash
sudo nmtui
```

After configuring the address, restart networking and verify the interface:

```bash
sudo systemctl restart NetworkManager
ip addr
ip route
```

Configure hostname resolution consistently on the nodes through `/etc/hosts` or the cluster's name service.

Verify connectivity with:

```bash
getent hosts master
ping -c 3 master
```

Install the basic administration and development tools:

```bash
sudo dnf update -y
sudo dnf install -y vim wget curl git \
  bind-utils net-tools htop tmux tree \
  gcc gcc-c++ gcc-gfortran make cmake \
  python3 openssh-server
```

## 5. Create the Login and Compute Nodes

Shut down the prepared virtual machine and create a snapshot. Create full clones for the login and compute nodes.

For each clone:

1. Assign a unique hostname.
2. Regenerate its machine ID.
3. Assign a unique internal IP address.
4. Configure SSH port forwarding if host-to-VM SSH access is required.
5. Verify networking and hostname resolution.

Regenerate the machine ID on each clone:

```bash
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
```

Set the appropriate hostname:

```bash
sudo hostnamectl set-hostname login.local
```

Use the corresponding hostname on each compute node.

## 6. Configure SSH Access

Passwordless SSH allows authorized cluster operations and supports workflows that need to launch processes across nodes.

Generate an SSH key for the intended user:

```bash
ssh-keygen -t ed25519
```

Copy the public key to the required nodes:

```bash
ssh-copy-id hpcadmin@master
ssh-copy-id hpcadmin@compute01
ssh-copy-id hpcadmin@compute02
ssh-copy-id hpcadmin@compute03
```

Test the connections:

```bash
ssh master hostname
ssh compute01 hostname
ssh compute02 hostname
ssh compute03 hostname
```

## 7. Configure Shared Storage with NFS

NFS provides shared directories so users can access the same files from the login and compute nodes.

### 7.1 Install NFS packages

Install the NFS utilities on the participating nodes:

```bash
sudo dnf install -y nfs-utils
```

### 7.2 Configure the NFS server

On the master node, enable the NFS server:

```bash
sudo systemctl enable --now nfs-server
systemctl status nfs-server
```

Configure `/etc/exports`:

```text
/home 10.10.10.0/24(rw,sync,no_root_squash)
```

Apply and verify the exports:

```bash
sudo exportfs -rav
sudo exportfs -v
```

### 7.3 Configure NFS clients

On each client node, create the mount point and mount the shared home directory:

```bash
sudo mkdir -p /home
sudo mount master:/home /home
df -h
```

Configure `/etc/fstab` for persistent mounting:

```text
master:/home /home nfs defaults 0 0
```

Validate the mount configuration:

```bash
sudo mount -a
mount | grep /home
```

Create a test file from one node and verify that it is visible from another node.

## 8. Configure Munge Authentication and SLURM

SLURM manages compute resources and schedules submitted jobs. Munge provides authentication between participating cluster services.

### 8.1 Install the required packages

Perform these steps on the master, login, and compute nodes.

Enable the required repositories on the Rocky Linux installation:

```bash
sudo dnf install -y epel-release
sudo crb enable
```

Install Munge and SLURM packages:

```bash
sudo dnf install -y \
  munge munge-libs \
  slurm slurm-slurmd slurm-slurmctld
```

Verify the installed packages and service accounts:

```bash
rpm -qa | grep -E 'munge|slurm'
id munge
id slurm
```

If the `slurm` account is missing, the reference procedure creates it as a system account:

```bash
sudo groupadd slurm
sudo useradd -r -g slurm -s /sbin/nologin slurm
```

Check first that the group and account do not already exist before creating them.

### 8.2 Generate the Munge key on the master

Munge uses a shared secret key to authenticate credentials between cluster nodes. **Every participating node must use the same `/etc/munge/munge.key`.**

Run this command on the master only:

```bash
sudo /usr/sbin/create-munge-key
```

Check that the key exists:

```bash
sudo ls -l /etc/munge/munge.key
```

### 8.3 Distribute the Munge key

Copy the key securely from the master to the other participating nodes. The reference procedure stages it in `/tmp/munge.key` and then installs it on each node.

For example, from the master:

```bash
sudo scp /etc/munge/munge.key \
  login:/tmp/munge.key

sudo scp /etc/munge/munge.key \
  compute01:/tmp/munge.key

sudo scp /etc/munge/munge.key \
  compute02:/tmp/munge.key

sudo scp /etc/munge/munge.key \
  compute03:/tmp/munge.key
```

On each receiving node, stop Munge before replacing the key:

```bash
sudo systemctl stop munge
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

Start and enable Munge on **all participating nodes**:

```bash
sudo systemctl enable --now munge
systemctl status munge
```

Verify local credential creation:

```bash
munge -n | unmunge
```

Then test authentication from the master to the other nodes:

```bash
munge -n | ssh login unmunge
munge -n | ssh compute01 unmunge
munge -n | ssh compute02 unmunge
munge -n | ssh compute03 unmunge
```

A successful result should include `STATUS: Success`.

Never commit `/etc/munge/munge.key` to GitHub or share it publicly.

### 8.4 Create `/etc/slurm/slurm.conf`

SLURM uses `/etc/slurm/slurm.conf` to define the cluster, controller, compute nodes, authentication method, scheduler, and partitions. The configuration must be consistent across all participating nodes.

On the master:

```bash
sudo mkdir -p /etc/slurm
sudo vim /etc/slurm/slurm.conf
```

Use the following configuration from the installation reference for the example three-compute-node lab:

```ini
ClusterName=hpccluster
SlurmctldHost=master

MpiDefault=none
ProctrackType=proctrack/cgroup
ReturnToService=2

SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid

SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld

SlurmUser=slurm

SwitchType=switch/none
TaskPlugin=task/affinity

SchedulerType=sched/backfill

SelectType=select/cons_tres
SelectTypeParameters=CR_Core

AuthType=auth/munge

NodeName=compute01 CPUs=2 State=UNKNOWN
NodeName=compute02 CPUs=2 State=UNKNOWN
NodeName=compute03 CPUs=2 State=UNKNOWN

PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

**Configuration values explained:**

| Setting | Purpose |
|---|---|
| `ClusterName` | Names the SLURM cluster |
| `SlurmctldHost` | Identifies the controller host |
| `AuthType=auth/munge` | Uses Munge for authentication |
| `NodeName` | Defines each compute node and its configured CPU count |
| `SchedulerType` | Selects the backfill scheduler |
| `SelectType` | Selects the resource-selection plugin |
| `PartitionName` | Defines a queue of nodes available for jobs |
| `StateSaveLocation` | Stores controller state |
| `SlurmdSpoolDir` | Stores compute-daemon state |

The node names and CPU counts must match the actual machines. This example belongs to the reference lab; use the separate configuration example below when documenting the different xCAT-based cluster.

### 8.5 Create the SLURM runtime directories

On the master:

```bash
sudo mkdir -p /var/spool/slurmctld
sudo mkdir -p /var/spool/slurmd
sudo chown slurm:slurm /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmd
```

On each compute node:

```bash
sudo mkdir -p /var/spool/slurmd
sudo chown slurm:slurm /var/spool/slurmd
```

Verify the directory ownership:

```bash
ls -ld /var/spool/slurmctld
ls -ld /var/spool/slurmd
```

### 8.6 Distribute the SLURM configuration

Copy the configuration from the master to the other nodes:

```bash
scp /etc/slurm/slurm.conf login:/tmp/slurm.conf
scp /etc/slurm/slurm.conf compute01:/tmp/slurm.conf
scp /etc/slurm/slurm.conf compute02:/tmp/slurm.conf
scp /etc/slurm/slurm.conf compute03:/tmp/slurm.conf
```

On each receiving node, install it in the SLURM configuration directory:

```bash
sudo mv /tmp/slurm.conf /etc/slurm/slurm.conf
```

Confirm that `/etc/slurm/slurm.conf` is consistent across the cluster.

### 8.7 Start the SLURM controller and node daemons

On the master only, start the controller:

```bash
sudo systemctl enable --now slurmctld
systemctl status slurmctld
```

On each compute node, start the compute daemon:

```bash
sudo systemctl enable --now slurmd
systemctl status slurmd
```

The login node is used for user access and job submission. Start `slurmd` there only if it is intended to act as a compute node in your configuration.

### 8.8 Verify and reconfigure SLURM

On the master:

```bash
sudo scontrol reconfigure
sinfo
scontrol show nodes
squeue
```

If a service fails to start, inspect its logs:

```bash
sudo journalctl -u slurmctld --no-pager -n 50
sudo journalctl -u slurmd --no-pager -n 50
```

If a compute node reports a CPU or hardware-topology mismatch, compare the node's actual hardware with the `CPUs` value in `slurm.conf` before changing its state. Do not delete spool state or resume a drained node without first diagnosing the cause.

### 8.9 Test job execution

From the login node, run a simple test:

```bash
srun hostname
```

Check the scheduler's node and partition status:

```bash
sinfo
scontrol show nodes
```

A successful test should run on a configured compute node. If the node is unavailable, check hostname resolution, Munge authentication, SLURM configuration consistency, daemon status, and the reported node state.

### SLURM configuration references

- [SLURM configuration example in this repository](../configs/slurm/slurm.conf.example)
- [SLURM configuration notes in this repository](../configs/slurm/README.md)



## 9. Configure the Shared Software Filesystem

A shared `/apps` filesystem provides a common location for compilers, libraries, and scientific applications.

1. Attach an additional virtual disk to the master.
2. Identify the correct disk device.
3. Create a partition and an XFS filesystem.
4. Mount the filesystem at `/apps`.
5. Configure the persistent mount in `/etc/fstab`.
6. Create the software directory hierarchy.
7. Export `/apps` through NFS.
8. Mount `/apps` on the required client nodes.
9. Verify that the same files are visible across the cluster.

## 10. Configure Centralized User Management with LDAP

OpenLDAP provides a centralized directory for user and group information. Client services allow Linux systems to resolve those identities.

### 10.1 Install the LDAP server

On the master node:

```bash
sudo dnf install -y openldap \
  openldap-servers openldap-clients
sudo systemctl enable --now slapd
systemctl status slapd
```

Configure the LDAP database, directory suffix, administrator identity, and password hash.

### 10.2 Create directory entries

Create the LDAP directory structure, including the `People` and `Groups` organizational units.

Add a group and a user entry with unique UID and GID values, a home-directory path, and a login shell.

Use `ldapadd` to add entries and `ldapsearch` to verify them.

### 10.3 Configure LDAP clients

Install the LDAP client packages on the login and compute nodes.

Configure `/etc/nslcd.conf` with the LDAP server and search base:

```text
uid nslcd
gid ldap
uri ldap://master
base dc=hpc,dc=local
```

Restrict the configuration file's permissions and enable the service:

```bash
sudo chmod 600 /etc/nslcd.conf
sudo systemctl enable --now nslcd
systemctl status nslcd
```

Configure NSS to use LDAP for user and group lookups, then restart `nslcd`.

### 10.4 Verify centralized user lookup

Test the directory query:

```bash
ldapsearch -x -H ldap://master \
  -b "dc=hpc,dc=local" uid=testUser
```

Verify that Linux recognizes the user:

```bash
getent passwd testUser
id testUser
```

Configure PAM and automatic home-directory creation as required by the installed Rocky Linux authentication stack.

## 11. Automate LDAP User Administration

User-management scripts can automate the creation, deletion, and password updates of LDAP users.

After each operation:

- Verify the LDAP entry.
- Check user and group lookup from a client node.
- Confirm the home-directory behavior.
- Handle credentials securely.

## 12. Final Validation

Verify each major component before considering the installation complete.

- [ ] Hostnames and internal network connectivity work.
- [ ] SSH access between the required nodes works.
- [ ] NFS exports and mounts are available.
- [ ] Munge authentication succeeds.
- [ ] SLURM controller and compute daemons are active.
- [ ] `sinfo` reports the configured nodes and partitions.
- [ ] A test SLURM job completes and produces output.
- [ ] Shared `/apps` storage is available where required.
- [ ] LDAP user and group lookups work on client nodes.

## Reference

The installation topics in this document are summarized and reorganized from the supplied *HPC Mini Cluster Documentation* PDF. The VirtualBox lab topology and the separate xCAT/SLURM configuration examples in this repository represent different documented environments.
