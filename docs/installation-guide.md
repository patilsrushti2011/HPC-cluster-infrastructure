# HPC Cluster Setup Guide: SLURM, NFS, and LDAP

This guide extracts the SLURM, NFS, and LDAP portions of the supplied
*HPC_mini_cluster_docs.pdf* and organizes them for the `docs/` folder of
a GitHub mini-HPC-cluster project.

It documents the main setup flow, important configuration files,
validation commands, and troubleshooting checks. It is a lab guide, not
a universal copy-and-paste installer.

> **Important --- adapt this guide to your cluster.** The source
> document uses the hostnames `master`, `login`, `compute01`,
> `compute02`, and `compute03`, with the internal network
> `10.10.10.0/24` and LDAP base `dc=hpc,dc=local`. Your xCAT cluster may
> use different hostnames, IP addresses, node roles, and
> operating-system versions. Replace example values with your actual
> configuration before running commands.

> **Security:** Never commit passwords, LDAP password hashes,
> `/etc/munge/munge.key`, private SSH keys, or other secrets. Use
> sanitized `.example` configuration files in a public repository. Back
> up authentication configuration before editing it.

## 1. Example cluster roles

The supplied guide describes these roles:

  -----------------------------------------------------------------------
  Node                                Main role
  ----------------------------------- -----------------------------------
  Master                              SLURM controller, NFS server, LDAP
                                      server, cluster management

  Login node                          User access, compilation, job
                                      submission, NFS and LDAP client

  Compute nodes                       Execute scheduled jobs; run
                                      `slurmd`; use shared NFS storage
                                      and LDAP identities
  -----------------------------------------------------------------------

In a cluster managed by xCAT, provisioning and node-management
responsibilities may be placed differently. Document the roles that are
actually configured in your environment.

## 2. NFS shared storage

### 2.1 Purpose

NFS makes shared directories available on multiple cluster nodes. The
source guide exports `/home` for user home directories and later exports
`/apps` for shared HPC software.

The expected outcome is that a file created in a shared directory from
one node can be read from another node.

### 2.2 Install NFS utilities

Run on the server and client nodes:

``` bash
sudo dnf install -y nfs-utils
```

### 2.3 Configure the NFS server

Run on the node that hosts the NFS share.

Enable and start the NFS server:

``` bash
sudo systemctl enable --now nfs-server
systemctl status nfs-server
```

The source guide configures `/home` in `/etc/exports`:

``` text
/home 10.10.10.0/24(rw,sync,no_root_squash)
```

If your cluster also shares `/apps`, the source guide adds:

``` text
/apps 10.10.10.0/24(rw,sync,no_root_squash)
```

These are examples from the source guide, not values to use blindly.
Replace the network and paths with your own.

-   `rw`: allows read/write access.
-   `sync`: asks the server to confirm writes synchronously.
-   `no_root_squash`: preserves root privileges from the client and can
    create a security risk. Use it only when you understand and
    explicitly require that behavior. Prefer safer export options for
    your environment.

Apply and inspect the exports:

``` bash
sudo exportfs -rav
sudo exportfs -v
showmount -e localhost
```

### 2.4 Mount the share on clients

On each intended client node, create the mount point if needed:

``` bash
sudo mkdir -p /home
```

The source guide moves an existing local `/home` to `/home.local` before
mounting. **Do not run this command without checking your system
first**; it can affect local user data and active home directories.

``` bash
# Source-guide example only; review local data before doing this.
sudo mv /home /home.local
sudo mkdir -p /home
sudo mount master:/home /home
```

For a shared software directory, the source guide uses `/apps`:

``` bash
sudo mkdir -p /apps
sudo mount master:/apps /apps
```

Check the mounted filesystems:

``` bash
df -hT
findmnt /home
findmnt /apps
```

### 2.5 Make NFS mounts persistent

On client nodes, add the appropriate entries to `/etc/fstab`. The source
guide uses:

``` text
master:/home  /home  nfs  defaults  0 0
master:/apps  /apps  nfs  defaults,_netdev  0 0
```

Only add the entries for shares you actually export. Test the file
before rebooting:

``` bash
sudo mount -a
```

If this reports an error, correct it before rebooting. Confirm the mount
is restored after a controlled reboot, once the NFS server is available.

### 2.6 Validate shared storage

Create a harmless test file from one client:

``` bash
touch ~/shared_test
```

On another node, verify that the same file is visible:

``` bash
ls -l ~/shared_test
```

For `/apps`, the source guide tests by creating a file on the server and
listing it from a client:

``` bash
# On the NFS server
touch /apps/testfile

# On a client
ls -l /apps/testfile
```

**Evidence for GitHub:** a sanitized `/etc/exports` example, relevant
`/etc/fstab` example, `exportfs -v` output, `df -hT`/`findmnt` output,
and a screenshot showing the shared-file test.

## 3. SLURM workload scheduling and Munge

### 3.1 Purpose and service roles

SLURM schedules jobs and allocates compute resources.

  Node role       Services in the source guide
  --------------- ----------------------------------
  Master          `munged`, `slurmctld`
  Login           `munged`; users submit jobs here
  Compute nodes   `munged`, `slurmd`

The source guide states that `/etc/slurm/slurm.conf` must be consistent
across the participating nodes. The login node is not a compute node in
that example and should not run `slurmd` unless you explicitly
configured it as a compute node.

### 3.2 Install packages

The source guide uses Rocky Linux 9 package commands. Confirm repository
and package availability for your exact OS release before running them.

On all participating nodes, according to the source guide:

``` bash
sudo dnf install -y epel-release
sudo crb enable
sudo dnf install -y munge munge-libs slurm slurm-slurmd slurm-slurmctld
```

Verify installed packages and service accounts:

``` bash
rpm -qa | grep -E 'munge|slurm'
id munge
id slurm
```

The source guide notes that if the `slurm` system account is missing, it
may need to be created according to the packaging and local
installation. Do not create duplicate accounts; check the package and
existing account state first.

### 3.3 Configure Munge authentication

Munge provides authentication between cluster services. All
participating nodes must use the same trusted Munge key.

On the designated controller, the source guide generates a key with:

``` bash
sudo /usr/sbin/create-munge-key
```

Check ownership and permissions:

``` bash
sudo ls -l /etc/munge/munge.key
```

Distribute the key to the other participating nodes using a secure
administrative method. **Do not put the key in GitHub, a public folder,
or an ordinary project archive.** Do not leave temporary copies with
permissive permissions.

On each node, ensure the key is owned by `munge:munge` and is readable
only by the required account. Then enable Munge:

``` bash
sudo systemctl enable --now munge
systemctl status munge
```

Test authentication from the controller to a compute node:

``` bash
munge -n | ssh compute01 unmunge
```

Repeat for the other configured nodes. A successful decode should report
a success status. Use your real node names.

### 3.4 Create and distribute `slurm.conf`

The source guide creates `/etc/slurm/slurm.conf` on the master. Its
example is:

``` text
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

This is the source document's example, **not a ready-to-use
configuration for every cluster**. Before using it, update at least:

-   `ClusterName` and `SlurmctldHost`.
-   `NodeName` entries to match actual compute-node names.
-   `CPUs` to match the CPU resources configured and detected on each
    VM.
-   The partition name and limits to match your intended policy.
-   Any paths or plugin settings required by your installed SLURM
    version and local configuration.

Create the required runtime directories and ensure their ownership
matches the service account and your installation:

``` bash
# Example paths from the source guide
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd
sudo chown slurm:slurm /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmd
```

Create `/var/spool/slurmd` with the appropriate ownership on each
compute node as well.

Distribute the same reviewed `slurm.conf` to all nodes that need it,
placing it at the configuration path used by your installation.

### 3.5 Start services

On the controller:

``` bash
sudo systemctl enable --now slurmctld
systemctl status slurmctld
```

On each compute node:

``` bash
sudo systemctl enable --now slurmd
systemctl status slurmd
```

Check cluster status on a node with the SLURM client tools:

``` bash
sinfo
scontrol show nodes
squeue
```

The source guide's example uses the `debug` partition and compute nodes
named `compute01` through `compute03`. Your output will differ if your
names and partition differ.

### 3.6 Test job execution

From the login node, the source guide tests one node:

``` bash
srun hostname
```

It then tests multiple nodes:

``` bash
srun -N 3 hostname
srun -N 3 -n 6 hostname
srun -N 1 -n 2 hostname
```

Change the node and task counts to values supported by your cluster. The
commands demonstrate allocation and task placement; use your MPI job
script as an additional test if MPI is part of your project.

**Evidence for GitHub:** sanitized `slurm.conf.example`, `sinfo`,
`squeue`, `scontrol show nodes`, a successful job script, and the job
output. Never include the Munge key.

## 4. LDAP centralized identity management

### 4.1 Purpose and roles

The source guide uses OpenLDAP with `nslcd`, NSS, and PAM.

  -----------------------------------------------------------------------
  Node role                           LDAP configuration
  ----------------------------------- -----------------------------------
  Master                              OpenLDAP server (`slapd`)

  Login node                          LDAP client

  Compute nodes                       LDAP clients

  Master/controller                   Must also resolve LDAP identities
                                      if SLURM runs there under the
                                      source architecture
  -----------------------------------------------------------------------

LDAP centralizes account information so that the cluster can resolve the
same user and group IDs on multiple nodes. Matching UID/GID values are
important when users access shared NFS files.

The source guide uses the example LDAP base DN `dc=hpc,dc=local`, group
`hpcusers`, and test user `testUser`. Replace these with your own
directory design.

### 4.2 Install and start OpenLDAP on the server

Run only on the designated LDAP server:

``` bash
sudo dnf install -y openldap openldap-servers openldap-clients
sudo systemctl enable --now slapd
systemctl status slapd
```

Generate an administrator password hash interactively:

``` bash
slappasswd
```

Do not publish the password or generated hash. The source guide stores
the hash in the LDAP database configuration and uses the directory base
`dc=hpc,dc=local`, with manager DN `cn=Manager,dc=hpc,dc=local`.

The source guide's database LDIF example uses these attributes:

``` text
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=hpc,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=hpc,dc=local

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: PASTE_HASH_HERE
```

The database index (`{2}mdb`) may differ on another installation.
Inspect your server configuration and adapt the LDIF rather than
assuming that index is correct.

Apply the configuration using the local LDAP administration method
described in the source guide:

``` bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f db.ldif
```

Import the schemas shown in the source guide if they are not already
present:

``` bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

Schema imports can fail if a schema is already loaded; check the
existing configuration before repeating them.

### 4.3 Create the LDAP directory tree, group, and test user

The source guide creates a base tree with `ou=People` and `ou=Groups`
under `dc=hpc,dc=local`.

Example `base.ldif`:

``` text
dn: dc=hpc,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: HPC Cluster
dc: hpc

dn: ou=People,dc=hpc,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=hpc,dc=local
objectClass: organizationalUnit
ou: Groups
```

Add the base tree:

``` bash
ldapadd -x -D "cn=Manager,dc=hpc,dc=local" -W -f base.ldif
```

The source guide defines a POSIX group with GID `10000`:

``` text
dn: cn=hpcusers,ou=Groups,dc=hpc,dc=local
objectClass: posixGroup
cn: hpcusers
gidNumber: 10000
```

It then creates a test user with UID `10001`, primary GID `10000`, home
`/home/testUser`, and shell `/bin/bash`. Generate the user's password
hash interactively with `slappasswd` and put the hash in a local LDIF
file that is **not committed to GitHub**.

Example user entry structure:

``` text
dn: uid=testUser,ou=People,dc=hpc,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Test User
sn: User
uid: testUser
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/testUser
loginShell: /bin/bash
userPassword: PASTE_HASH_HERE
```

Add the group and user with the source guide's pattern:

``` bash
ldapadd -x -D "cn=Manager,dc=hpc,dc=local" -W -f group.ldif
ldapadd -x -D "cn=Manager,dc=hpc,dc=local" -W -f user.ldif
```

These LDIF files are templates. Check your LDAP schema, directory
structure, UID/GID allocation, and password-handling policy before
applying them.

Verify the directory and user:

``` bash
ldapsearch -x -b "dc=hpc,dc=local"
ldapsearch -x -b "dc=hpc,dc=local" uid=testUser
```

### 4.4 Configure LDAP clients

The source guide installs the client packages on the login node, then
repeats the appropriate client setup on the other nodes:

``` bash
sudo dnf install -y openldap-clients nss-pam-ldapd
```

Example `/etc/nslcd.conf` from the source guide:

``` text
uid nslcd
gid ldap

uri ldap://master
base dc=hpc,dc=local
```

Set restrictive permissions on the client configuration:

``` bash
sudo chmod 600 /etc/nslcd.conf
sudo systemctl enable --now nslcd
systemctl status nslcd
```

If your configuration includes bind credentials, keep them out of GitHub
and protect the file.

Test connectivity from the client:

``` bash
ldapsearch -x -H ldap://master -b "dc=hpc,dc=local" uid=testUser
```

The source guide updates `/etc/nsswitch.conf` so Linux checks local
account files and LDAP:

``` text
passwd: files ldap
group:  files ldap
shadow: files ldap
```

Exact NSS and PAM configuration depends on the operating-system release
and authentication stack. On modern Rocky Linux, use the supported
system authentication tools and back up existing files before changes.
Do not blindly replace PAM files or remove existing `pam_unix.so`
entries; a bad PAM change can prevent logins.

The source guide uses `oddjob-mkhomedir` to support automatic
home-directory creation:

``` bash
sudo dnf install -y oddjob-mkhomedir
sudo systemctl enable --now oddjobd
```

Configure automatic home creation using the supported PAM/authentication
profile for your OS, rather than copying PAM lines without checking the
existing stack.

### 4.5 Verify LDAP identity and authentication

Run these checks on the master/controller, login node, and compute nodes
where LDAP client resolution is required:

``` bash
getent passwd testUser
id testUser
systemctl status nslcd
```

The source guide's sample account resolves as UID `10001` and GID
`10000`. Your output should match the actual UID/GID assigned in your
directory.

Test an LDAP bind interactively:

``` bash
ldapwhoami -x \
  -D "uid=testUser,ou=People,dc=hpc,dc=local" \
  -W \
  -H ldap://master
```

Test login with a dedicated non-administrative test account only after
client authentication and home-directory setup are complete.

**Evidence for GitHub:** sanitized client configuration examples,
redacted LDAP search output, `getent passwd`/`id` output, and a
successful test login. Do not publish password hashes, bind passwords,
or private user information.

## 5. SLURM + LDAP + NFS: important integration checks

These services depend on each other:

1.  LDAP provides consistent user and group identities.
2.  NFS provides shared home directories and, optionally, shared
    software.
3.  SLURM uses operating-system user identities when scheduling and
    launching jobs.

Check that the same LDAP user resolves with the same UID/GID on the
controller, login node, and compute nodes. Check that the user's home
directory is available through the intended NFS mount.

If jobs fail for LDAP users, the source guide specifically recommends
checking identity resolution on all relevant nodes:

``` bash
getent passwd testUser
id testUser
```

It also recommends checking the LDAP client service and SLURM services:

``` bash
systemctl status nslcd
systemctl status slurmctld
systemctl status slurmd
journalctl -u slurmctld -n 50
```

Use `slurmctld` checks on the controller and `slurmd` checks on compute
nodes.

## 6. Suggested GitHub files

For a repository, keep the real configuration private where necessary
and publish only reviewed examples:

``` text
configs/
├── nfs/
│   ├── exports.example
│   ├── fstab.example
│   └── README.md
├── slurm/
│   ├── slurm.conf.example
│   └── README.md
└── ldap/
    ├── nslcd.conf.example
    ├── base.ldif.example
    ├── group.ldif.example
    ├── user.ldif.example
    └── README.md

docs/
├── installation-guide.md
├── testing-and-validation.md
└── troubleshooting.md
```

Do not place `munge.key`, real LDAP password hashes, private keys, or
credentials in these folders. For LDAP examples, use clearly fake test
values and explain which values must be customized.

## 7. Final validation checklist

-   [ ] NFS server is active and exports the intended directories.
-   [ ] NFS clients mount the correct shares and shared-file visibility
    is tested.
-   [ ] NFS mounts survive a controlled reboot.
-   [ ] Munge is active and authentication works between the relevant
    nodes.
-   [ ] SLURM controller and compute daemons are active on the correct
    nodes.
-   [ ] `sinfo` and `squeue` show the expected cluster state.
-   [ ] A simple SLURM job completes successfully.
-   [ ] LDAP user lookup works on every node that needs the identity.
-   [ ] UID/GID values are consistent across nodes.
-   [ ] LDAP user authentication and home-directory access work.
-   [ ] A SLURM job submitted by the test user can access the expected
    NFS home directory.

## Source note

This guide is based on the supplied *HPC_mini_cluster_docs.pdf*,
particularly its sections on NFS shared storage (Section 10 and the
`/apps` filesystem in Section 12), Munge and SLURM (Section 11), and
OpenLDAP with `nslcd` (Section 14). It is reorganized for a GitHub
project and includes reminders to adapt sample hostnames, addresses, and
security-sensitive settings to the actual cluster.
