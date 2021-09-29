---
title: "Create Pacemaker Cluster"
date: 2021-09-23
draft: true
---

# Installation of pacemaker for HANA

## Prerequisite

2 cluster nodes must have :

* the same level of patchs (kernel, packages, ...)
* the same number of intalled packages
* the same number of enabled and disabled services
* 3 networks interfaces (for Public, for Replication and for High Availability)
* for the stonith : at least 2 Luns of 64mb for the storage

## Pacemaker packages Installation on both nodes

To install the HA stack on both cluster nodes, you need to install the pattern ha_sles and the resource agents SAPHanaSR

### On node 1

Install the ha_sles pattern :

```
node01:~ # zypper in -t pattern ha_sles
Refreshing service 'SUSE_Linux_Enterprise_Server_for_SAP_Applications_12_SP1_x86_64'.
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 77 NEW packages are going to be installed:
  cluster-glue conntrack-tools corosync crmsh crmsh-scripts csync2 ctdb dlm-kmp-default drbd
  drbd-kmp-default drbd-utils fence-agents graphviz graphviz-gd graphviz-gnome
  ha-cluster-bootstrap hawk2 ipvsadm ldirectord libcorosync4 libdlm libdlm3 libglue2 libnet9
  libnetfilter_cthelper0 libnetfilter_cttimeout1 libnetfilter_queue1 libpacemaker3 libqb0
  librsync1 lvm2-clvm lvm2-cmirrord ocfs2-kmp-default ocfs2-tools pacemaker pacemaker-cli
  patterns-ha-ha_sles perl-Encode-Locale perl-File-Listing perl-HTML-Parser perl-HTML-Tagset
  perl-HTTP-Cookies perl-HTTP-Daemon perl-HTTP-Date perl-HTTP-Message perl-HTTP-Negotiate
  perl-IO-HTML perl-IO-Socket-INET6 perl-IO-Socket-SSL perl-LWP-MediaTypes
  perl-LWP-Protocol-https perl-MailTools perl-Net-HTTP perl-Net-SSLeay perl-Socket6
  perl-TimeDate perl-WWW-RobotRules perl-libwww-perl pssh python-curses python-dateutil
  python-lxml python-parallax python-pexpect python-pssh python-suds release-notes-ha
  resource-agents ruby2.1-rubygem-bundler sbd sle-ha-guide_en-pdf sle-ha-manuals_en
  sle-ha-nfs-quick_en-pdf tdb-tools yast2-cluster yast2-drbd yast2-iplb

The following NEW pattern is going to be installed:
  ha_sles

The following 15 recommended packages were automatically selected:
  crmsh fence-agents graphviz-gd graphviz-gnome hawk2 libdlm ocfs2-kmp-default ocfs2-tools
  perl-IO-Socket-SSL perl-LWP-Protocol-https perl-TimeDate sbd sle-ha-guide_en-pdf
  sle-ha-manuals_en sle-ha-nfs-quick_en-pdf

77 new packages to install.
Overall download size: 44.5 MiB. Already cached: 0 B. After the operation, additional 102.8
MiB will be used.
Continue? [y/n/? shows all options] (y): y
Retrieving package dlm-kmp-default-4.0.2_k3.12.49_11-33.18.x86_64
                                                      (1/77),  93.0 KiB (298.2 KiB unpacked)
Retrieving: dlm-kmp-default-4.0.2_k3.12.49_11-33.18.x86_64.rpm .......................[done]
[...]
(72/77) Installing: pacemaker-cli-1.1.13-20.1.x86_64 .................................[done]
Additional rpm output:
Updating /etc/sysconfig/pacemaker...
Updating /etc/sysconfig/pacemaker...


(73/77) Installing: pacemaker-1.1.13-20.1.x86_64 .....................................[done]
(74/77) Installing: crmsh-2.2.1-23.2.noarch ..........................................[done]
(75/77) Installing: hawk2-1.0.1+git.1456406635.49e230d-18.1.x86_64 ...................[done]
(76/77) Installing: ha-cluster-bootstrap-0.4+git.1441350120.8e9abbe-7.1.noarch .......[done]
(77/77) Installing: patterns-ha-ha_sles-12-9.1.x86_64 ................................[done]
```

Now install the SAPHanaSR and documentation package :
```
node01:~ # zypper in SAPHanaSR SAPHanaSR-doc
Refreshing service 'SUSE_Linux_Enterprise_Server_for_SAP_Applications_12_SP1_x86_64'.
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 2 NEW packages are going to be installed:
  SAPHanaSR SAPHanaSR-doc

2 new packages to install.
Overall download size: 91.6 KiB. Already cached: 0 B. After the operation, additional 241.4
KiB will be used.
Continue? [y/n/? shows all options] (y): y
Retrieving package SAPHanaSR-0.152.21-1.1.noarch       (1/2),  50.2 KiB (206.0 KiB unpacked)
Retrieving: SAPHanaSR-0.152.21-1.1.noarch.rpm ........................................[done]
Retrieving package SAPHanaSR-doc-0.152.21-1.1.noarch   (2/2),  41.4 KiB ( 35.4 KiB unpacked)
Retrieving: SAPHanaSR-doc-0.152.21-1.1.noarch.rpm ....................................[done]
Checking for file conflicts: .........................................................[done]
(1/2) Installing: SAPHanaSR-0.152.21-1.1.noarch ......................................[done]
(2/2) Installing: SAPHanaSR-doc-0.152.21-1.1.noarch ..................................[done]
node02:~ # zypper in SAPHanaSR SAPHanaSR-doc
Refreshing service 'SUSE_Linux_Enterprise_Server_for_SAP_Applications_12_SP1_x86_64'.
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 2 NEW packages are going to be installed:
  SAPHanaSR SAPHanaSR-doc

2 new packages to install.
Overall download size: 91.6 KiB. Already cached: 0 B. After the operation, additional 241.4
KiB will be used.
Continue? [y/n/? shows all options] (y): y
Retrieving package SAPHanaSR-0.152.21-1.1.noarch       (1/2),  50.2 KiB (206.0 KiB unpacked)
Retrieving: SAPHanaSR-0.152.21-1.1.noarch.rpm ........................................[done]
Retrieving package SAPHanaSR-doc-0.152.21-1.1.noarch   (2/2),  41.4 KiB ( 35.4 KiB unpacked)
Retrieving: SAPHanaSR-doc-0.152.21-1.1.noarch.rpm ....................................[done]
Checking for file conflicts: .........................................................[done]
(1/2) Installing: SAPHanaSR-0.152.21-1.1.noarch ......................................[done]
(2/2) Installing: SAPHanaSR-doc-0.152.21-1.1.noarch ..................................[done]
```

### On node 2

Same operation as previous :

```
node02:~ # zypper in -t pattern ha_sles
Refreshing service 'SUSE_Linux_Enterprise_Server_for_SAP_Applications_12_SP1_x86_64'.
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 77 NEW packages are going to be installed:
  cluster-glue conntrack-tools corosync crmsh crmsh-scripts csync2 ctdb dlm-kmp-default drbd
  drbd-kmp-default drbd-utils fence-agents graphviz graphviz-gd graphviz-gnome
  ha-cluster-bootstrap hawk2 ipvsadm ldirectord libcorosync4 libdlm libdlm3 libglue2 libnet9
  libnetfilter_cthelper0 libnetfilter_cttimeout1 libnetfilter_queue1 libpacemaker3 libqb0
  librsync1 lvm2-clvm lvm2-cmirrord ocfs2-kmp-default ocfs2-tools pacemaker pacemaker-cli
  patterns-ha-ha_sles perl-Encode-Locale perl-File-Listing perl-HTML-Parser perl-HTML-Tagset
  perl-HTTP-Cookies perl-HTTP-Daemon perl-HTTP-Date perl-HTTP-Message perl-HTTP-Negotiate
  perl-IO-HTML perl-IO-Socket-INET6 perl-IO-Socket-SSL perl-LWP-MediaTypes
  perl-LWP-Protocol-https perl-MailTools perl-Net-HTTP perl-Net-SSLeay perl-Socket6
  perl-TimeDate perl-WWW-RobotRules perl-libwww-perl pssh python-curses python-dateutil
  python-lxml python-parallax python-pexpect python-pssh python-suds release-notes-ha
  resource-agents ruby2.1-rubygem-bundler sbd sle-ha-guide_en-pdf sle-ha-manuals_en
  sle-ha-nfs-quick_en-pdf tdb-tools yast2-cluster yast2-drbd yast2-iplb

The following NEW pattern is going to be installed:
  ha_sles

The following 15 recommended packages were automatically selected:
  crmsh fence-agents graphviz-gd graphviz-gnome hawk2 libdlm ocfs2-kmp-default ocfs2-tools
  perl-IO-Socket-SSL perl-LWP-Protocol-https perl-TimeDate sbd sle-ha-guide_en-pdf
  sle-ha-manuals_en sle-ha-nfs-quick_en-pdf

77 new packages to install.
Overall download size: 44.5 MiB. Already cached: 0 B. After the operation, additional 102.8
MiB will be used.
Continue? [y/n/? shows all options] (y): y
Retrieving package dlm-kmp-default-4.0.2_k3.12.49_11-33.18.x86_64
                                                      (1/77),  93.0 KiB (298.2 KiB unpacked)
Retrieving: dlm-kmp-default-4.0.2_k3.12.49_11-33.18.x86_64.rpm .......................[done]
[...]
(72/77) Installing: pacemaker-cli-1.1.13-20.1.x86_64 .................................[done]
Additional rpm output:
Updating /etc/sysconfig/pacemaker...
Updating /etc/sysconfig/pacemaker...


(73/77) Installing: pacemaker-1.1.13-20.1.x86_64 .....................................[done]
(74/77) Installing: crmsh-2.2.1-23.2.noarch ..........................................[done]
(75/77) Installing: hawk2-1.0.1+git.1456406635.49e230d-18.1.x86_64 ...................[done]
(76/77) Installing: ha-cluster-bootstrap-0.4+git.1441350120.8e9abbe-7.1.noarch .......[done]
(77/77) Installing: patterns-ha-ha_sles-12-9.1.x86_64 ................................[done]
```

Same for resource agent package :

```
node02:~ # zypper in SAPHanaSR SAPHanaSR-doc
Refreshing service 'SUSE_Linux_Enterprise_Server_for_SAP_Applications_12_SP1_x86_64'.
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 2 NEW packages are going to be installed:
  SAPHanaSR SAPHanaSR-doc

2 new packages to install.
Overall download size: 91.6 KiB. Already cached: 0 B. After the operation, additional 241.4
KiB will be used.
Continue? [y/n/? shows all options] (y): y
Retrieving package SAPHanaSR-0.152.21-1.1.noarch       (1/2),  50.2 KiB (206.0 KiB unpacked)
Retrieving: SAPHanaSR-0.152.21-1.1.noarch.rpm ........................................[done]
Retrieving package SAPHanaSR-doc-0.152.21-1.1.noarch   (2/2),  41.4 KiB ( 35.4 KiB unpacked)
Retrieving: SAPHanaSR-doc-0.152.21-1.1.noarch.rpm ....................................[done]
Checking for file conflicts: .........................................................[done]
(1/2) Installing: SAPHanaSR-0.152.21-1.1.noarch ......................................[done]
(2/2) Installing: SAPHanaSR-doc-0.152.21-1.1.noarch ..................................[done]
```

## Generate SSH key for the cluster

The pacemaker cluster need to communicate through the root user. It's necessary to generate them with the command line bellow :

```
node01:~ # ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
5d:42:1b:96:49:62:30:2e:c2:95:73:0c:98:4a:7e:9a [MD5] root@node01
The key's randomart image is:
+--[ RSA 2048]----+
|   oo+o.o.=o     |
| oo.o.oo +oo     |
|o.o .o.   o .    |
|.. o .   . o     |
|  +     S .      |
| E               |
|                 |
|                 |
|                 |
+--[MD5]----------+
```

From the end-user view, it's necessary to have the same ssh user and ssh host keys. For that, we need to copy files from node1 to node2.

### ssh user key

Copy from node1 to node2 of the ssh user key
```
node01:~ # scp -r .ssh node02:
Password:
known_hosts                                               100%  548     0.5KB/s   00:00
id_rsa                                                    100% 1675     1.6KB/s   00:00
id_rsa.pub                                                100%  397     0.4KB/s   00:00
```

Authorize the root's user key to connect without password
```
node01:~ # cp .ssh/id_rsa.pub .ssh/authorized_keys
node02:~ # cp .ssh/id_rsa.pub .ssh/authorized_keys
```

### ssh host key

Copy ssh_host key from node1 to node2
```
node01:/etc/ssh # scp ssh_host_* node02:/etc/ssh
ssh_host_dsa_key                                          100%  668     0.7KB/s   00:00
ssh_host_dsa_key.pub                                      100%  605     0.6KB/s   00:00
ssh_host_ecdsa_key                                        100%  227     0.2KB/s   00:00
ssh_host_ecdsa_key.pub                                    100%  177     0.2KB/s   00:00
ssh_host_ed25519_key                                      100%  411     0.4KB/s   00:00
ssh_host_ed25519_key.pub                                  100%   97     0.1KB/s   00:00
ssh_host_key                                              100%  980     1.0KB/s   00:00
ssh_host_key.pub                                          100%  645     0.6KB/s   00:00
ssh_host_rsa_key                                          100% 1675     1.6KB/s   00:00
ssh_host_rsa_key.pub                                      100%  397     0.4KB/s   00:00
```

## SBD

To have a supported cluster, it's mandatory to have a valid fence method. The method used is SBD.
First, we need to identify theses disks.

### Identify stonith devices


```
node01:~ # ll /dev/disk/by-id/scsi-1HITACHI*
lrwxrwxrwx 1 root root 9 Jan 16 15:36 /dev/disk/by-id/scsi-1HITACHI_5040A76500F0 -> ../../sdd
lrwxrwxrwx 1 root root 9 Jan 16 15:36 /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 -> ../../sde
node02:~ # ll /dev/disk/by-id/scsi-1HITACHI*
lrwxrwxrwx 1 root root 9 Jan 16 15:35 /dev/disk/by-id/scsi-1HITACHI_5040A76500F0 -> ../../sdd
lrwxrwxrwx 1 root root 9 Jan 16 15:35 /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 -> ../../sde
```

### Install watchdog on both nodes

Before running the initialization of the cluster with the ha-cluster-init command, it's necessary to load the watchdog module. This operation is needed on both nodes :

```
node01:~ # echo softdog > /etc/modules-load.d/watchdog.conf
node01:~ # modprobe softdog
node02:~ # echo softdog > /etc/modules-load.d/watchdog.conf
node02:~ # modprobe softdog
```

## Initialization of the cluster infrastructure

To initialize the cluster, we have to answer some question from the ha-cluster-init script.
* "Network address to bind to" is the network address of the HA Network (private network for corosync).
* multicast is not used, so valid the default answer. We will remove them after the end of the script.
* you can put only 1 sbd device to the question : "Path to storage device". We will add the second after the end of the script.

```
node01:~ # ha-cluster-init
  Enabling sshd.service
  /root/.ssh/id_rsa already exists - overwrite? [y/N] N
  Configuring csync2
  Generating csync2 shared key (this may take a while)...done
  Enabling csync2.socket
  csync2 checking files

Configure Corosync:
  This will configure the cluster messaging layer.  You will need
  to specify a network address over which to communicate (default
  is eth0's network, but you can use the network address of any
  active interface), a multicast address and multicast port.

  Network address to bind to (e.g.: 192.168.1.0) [10.117.158.0] 192.168.254.0
  Multicast address (e.g.: 239.x.x.x) [239.97.213.131]
  Multicast port [5405]

Configure SBD:
  If you have shared storage, for example a SAN or iSCSI target,
  you can use it avoid split-brain scenarios by configuring SBD.
  This requires a 1 MB partition, accessible to all nodes in the
  cluster.  The device path must be persistent and consistent
  across all nodes in the cluster, so /dev/disk/by-id/* devices
  are a good choice.  Note that all data on the partition you
  specify here will be destroyed.

  Do you wish to use SBD? [y/N] y
  Path to storage device (e.g. /dev/disk/by-id/...) [] /dev/disk/by-id/scsi-1HITACHI_5040A76500F0
  All data on /dev/disk/by-id/scsi-1HITACHI_5040A76500F0 will be destroyed
  Are you sure you wish to use this device [y/N] y
  Initializing SBD......done
  Enabling hawk.service
    HA Web Konsole is now running, to see cluster status go to:
      https://10.117.158.11:7630/
    Log in with username 'hacluster', password 'linux'
WARNING: You should change the hacluster password to something more secure!
  Enabling pacemaker.service
  Waiting for cluster........done
  Loading initial configuration

Configure Administration IP Address:
  Optionally configure an administration virtual IP
  address. The purpose of this IP address is to
  provide a single IP that can be used to interact
  with the cluster, rather than using the IP address
  of any specific cluster node.

  Do you wish to configure an administration IP? [y/N] N
  Done (log saved to /var/log/ha-cluster-bootstrap.log)
```

To add the second SBD device, you have to stop the pacemaker service and modify the sbd stored in /etc/sysconfig.

Stop pacemaker, and modify the configuration to done all modification.

```
node01:~ # service pacemaker stop
```
Create the new sbd device on the second LUN
```
node01:~ # sbd -d /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 create
Initializing device /dev/disk/by-id/scsi-1HITACHI_5040A76500F1
Creating version 2.1 header on device 4 (uuid: a3012040-64bd-4e9b-8260-47a0eae4d19d)
Initializing 255 slots on device 4
Device /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 is initialized.
node01:~ # sbd -d /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 allocate node01
Trying to allocate slot for node01 on device /dev/disk/by-id/scsi-1HITACHI_5040A76500F1.
slot 0 is unused - trying to own
Slot for node01 has been allocated on /dev/disk/by-id/scsi-1HITACHI_5040A76500F1.

node01:~ # sbd -d /dev/disk/by-id/scsi-1HITACHI_5040A76500F1 list
0       node01      clear
```

Edit the /etc/sysconfig/sbd configuration file to add the new Stonith device :
```
vi /etc/sysconfig/sbd
Change the line below
SBD_DEVICE="/dev/disk/by-id/scsi-1HITACHI_5040A76500F0"
to add the new Stonith device (separate with ;)
SBD_DEVICE="/dev/disk/by-id/scsi-1HITACHI_5040A76500F0; /dev/disk/by-id/scsi-1HITACHI_5040A76500F1"
```

Then edit /etc/corosync/corosync.conf and remove lines mcast.* inside the 'interface' and add the unicast configuration.
```
        interface {
                ringnumber:     0
                bindnetaddr:    192.168.254.0
                mcastaddr:      239.97.213.131 <- remove this line
                mcastport:      5405 <- remove this line
                ttl:            1
        }
        transport: udpu <- add this line
```

add node list between § totem and § logging
```
nodelist {
        node {
                ring0_addr: 192.168.254.14
        }
        node {
                ring0_addr: 192.168.254.15
        }
}
```

put the correct configuration for the quorum : remove  expected_votes: 1 and set two_node to 1
```
quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
        two_node: 1
}
```


## corosync.conf

This file is the result of all modification above.
```
node01:/etc/corosync # cat corosync.conf
# Please read the corosync.conf.5 manual page

totem {
        version:        2
        secauth:        off
        cluster_name:   hacluster
        clear_node_high_bit: yes

# Following are old corosync 1.4.x defaults from SLES
#       token:          5000
#       token_retransmits_before_loss_const: 10
#       join:           60
#       consensus:      6000
#       vsftype:        none
#       max_messages:   20
#       threads:        0

        crypto_cipher:  none
        crypto_hash:    none

        interface {
                ringnumber:     0
                bindnetaddr:    192.168.254.0
                ttl:            1
        }
        transport: udpu
}
nodelist {
        node {
                ring0_addr: 192.168.254.14
        }
        node {
                ring0_addr: 192.168.254.15
        }
}
logging {
        fileline:       off
        to_stderr:      no
        to_logfile:     no
        logfile:        /var/log/cluster/corosync.log
        to_syslog:      yes
        debug:          off
        timestamp:      on
        logger_subsys {
                subsys: QUORUM
                debug:  off
        }
}
quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
        two_node: 1
}
```

## Restart the pacemaker service on node 1

```
node01:/etc/corosync # service pacemaker start
```

## NODE 2 join the cluster

Run the ha-cluster-join script on the second node and enter the IP of the configured node.
No need to generate the ssh keys. It's already done.

```
node02:~ # ha-cluster-join

Join This Node to Cluster:
  You will be asked for the IP address of an existing node, from which
  configuration will be copied.  If you have not already configured
  passwordless ssh between nodes, you will be prompted for the root
  password of the existing node.

  IP address or hostname of existing node (e.g.: 192.168.1.1) [] 192.168.254.14
  Enabling sshd.service
  Retrieving SSH keys from 192.168.254.14
  /root/.ssh/id_rsa already exists - overwrite? [y/N] N
  No new SSH keys installed
  Configuring csync2
  Enabling csync2.socket
  Merging known_hosts
  Probing for new partitions......done
/usr/sbin/ha-cluster-join: line 209: [: -eq: unary operator expected
  Enabling hawk.service
    HA Web Konsole is now running, to see cluster status go to:
      https://10.117.158.17:7630/
    Log in with username 'hacluster', password 'linux'
WARNING: You should change the hacluster password to something more secure!
  Enabling pacemaker.service
  Waiting for cluster....done
  Done (log saved to /var/log/ha-cluster-bootstrap.log)
```



## Check the configuration

```
node01:~ # crm_mon -rADf1
Online: [ node01 node02 ]

Full list of resources:

 stonith-sbd    (stonith:external/sbd): Started node01

Node Attributes:
* Node node01:
* Node node02:

Migration Summary:
* Node node01:
* Node node02:
```

# Check hana version

Must be done by SAP Team, as sidadm

```
node01:~ # su - mfzadm
mfzadm@node01:/usr/sap/MFZ/HDB00> HDB version
HDB version info:
  version:             2.00.022.00.1511184640
  branch:              fa/hana2sp02
  git hash:            eeeca63c40ee9d73b6bdd41e42b1ded3bf7e16f7
  git merge time:      2017-11-20 14:30:40
  weekstone:           0000.00.0
  compile date:        2017-11-20 14:37:21
  compile host:        ld4552
  compile type:        rel
```

```
node02:~ # su - mfzadm
mfzadm@node02:/usr/sap/MFZ/HDB00> HDB version
HDB version info:
  version:             2.00.022.00.1511184640
  branch:              fa/hana2sp02
  git hash:            eeeca63c40ee9d73b6bdd41e42b1ded3bf7e16f7
  git merge time:      2017-11-20 14:30:40
  weekstone:           0000.00.0
  compile date:        2017-11-20 14:37:21
  compile host:        ld4552
  compile type:        rel
```

## Setting up Hana replication on master node

Must be done by SAP Team, as sidadm

```
mfzadm@node01:/usr/sap/MFZ> hdbnsutil -sr_enable --name=MFZ_node1
checking for active nameserver ...
nameserver is active, proceeding ...
successfully enabled system as system replication source site
done.
```

## Copy SSFS key on slave node

Must be done by SAP Team, as sidadm

```
node01:~ # scp /usr/sap/MFZ/SYS/global/security/rsecssfs/data/SSFS_MFZ.DAT node02:/usr/sap/MFZ/SYS/global/security/rsecssfs/data/
SSFS_MFZ.DAT                                              100% 2960     2.9KB/s   00:00
node01:~ # scp /usr/sap/MFZ/SYS/global/security/rsecssfs/key/SSFS_MFZ.KEY node02:/usr/sap/MFZ/SYS/global/security/rsecssfs/key
SSFS_MFZ.KEY                                              100%  187     0.2KB/s   00:00
```

## Modify global.ini to add ip address and node name used by HSR

Must be done by SAP Team, as sidadm

```
mfzadm@node01:/usr/sap/MFZ> cdcoc
mfzadm@node01:/usr/sap/MFZ/SYS/global/hdb/custom/config> vi global.ini
[...]

[system_replication_hostname_resolution]
192.168.253.14 = node01
192.168.253.15 = node02
```

```
mfzadm@node02:/usr/sap/MFZ> cdcoc
mfzadm@node02:/usr/sap/MFZ/SYS/global/hdb/custom/config> vi global.ini
[...]

[system_replication_hostname_resolution]
192.168.253.14 = node01
192.168.253.15 = node02
```

## Activate Hana replication on slave node

Must be done by SAP Team, as sidadm

```
mfzadm@node02:/usr/sap/MFZ> HDB stop
hdbdaemon will wait maximal 300 seconds for NewDB services finishing.
Stopping instance using: /usr/sap/MFZ/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function Stop 400

17.01.2018 09:34:14
Stop
OK
Waiting for stopped instance using: /usr/sap/MFZ/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function WaitforStopped 600 2


17.01.2018 09:34:48
WaitforStopped
OK
hdbdaemon is stopped.
```

Must be done by SAP Team, as sidadm

```
mfzadm@node02:/usr/sap/MFZ> hdbnsutil -sr_register --remoteHost=node01 --remoteInstance=00 --replicationMode=sync --name=MFZ_node2 --operationMode=delta_datashipping --force_full_replica
adding site ...
checking for inactive nameserver ...
nameserver node02:30001 not responding.
collecting information ...
updating local ini files ...
done.
```

Must be done by SAP Team, as sidadm

```
mfzadm@node02:/usr/sap/MFZ/SYS/global/hdb/custom/config> HDB start


StartService
Impromptu CCC initialization by 'rscpCInit'.
  See SAP note 1266393.
OK
OK
Starting instance using: /usr/sap/MFZ/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function StartWait 2700 2


17.01.2018 09:48:07
Start
OK

17.01.2018 09:48:57
StartWait
OK
```


## Check Hana replication on master node [during the force_full_replica]

```
node01:~ # su - mfzadm
mfzadm@node01:/usr/sap/MFZ> cdpy
mfzadm@node01:/usr/sap/MFZ/HDB00> cdpy
mfzadm@node01:/usr/sap/MFZ/HDB00/exe/python_support> python systemReplicationStatus.py
| Database | Host       | Port  | Service Name | Volume ID | Site ID | Site Name | Secondary  | Secondary | Secondary | Secondary | Secondary     | Replication | Replication  | Replication                         |
|          |            |       |              |           |         |           | Host       | Port      | Site ID   | Site Name | Active Status | Mode        | Status       | Status Details                      |
| -------- | ---------- | ----- | ------------ | --------- | ------- | --------- | ---------- | --------- | --------- | --------- | ------------- | ----------- | ------------ | ----------------------------------- |
| SYSTEMDB | node01 | 30001 | nameserver   |         1 |       1 | MFZ_node1 | node02 |     30001 |         2 | MFZ_node2 | YES           | SYNC        | ACTIVE       |                                     |
| MTD      | node01 | 30043 | indexserver  |         2 |       1 | MFZ_node1 | node02 |     30043 |         2 | MFZ_node2 | YES           | SYNC        | INITIALIZING | Full Replica: 47 % (40992/85488 MB) |

status system replication site "2": INITIALIZING
overall system replication status: INITIALIZING

Local System Replication State
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

mode: PRIMARY
site id: 1
site name: MFZ_node1
```

## Check Hana replication on master node [force_full_replica end]

```
mfzadm@node01:/usr/sap/MFZ/HDB00/exe/python_support> python systemReplicationStatus.py
| Database | Host       | Port  | Service Name | Volume ID | Site ID | Site Name | Secondary  | Secondary | Secondary | Secondary | Secondary     | Replication | Replication | Replication    |
|          |            |       |              |           |         |           | Host       | Port      | Site ID   | Site Name | Active Status | Mode        | Status      | Status Details |
| -------- | ---------- | ----- | ------------ | --------- | ------- | --------- | ---------- | --------- | --------- | --------- | ------------- | ----------- | ----------- | -------------- |
| SYSTEMDB | node01 | 30001 | nameserver   |         1 |       1 | MFZ_node1 | node02 |     30001 |         2 | MFZ_node2 | YES           | SYNC        | ACTIVE      |                |
| MTD      | node01 | 30043 | indexserver  |         2 |       1 | MFZ_node1 | node02 |     30043 |         2 | MFZ_node2 | YES           | SYNC        | ACTIVE      |                |

status system replication site "2": ACTIVE
overall system replication status: ACTIVE

Local System Replication State
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

mode: PRIMARY
site id: 1
site name: MFZ_node1
```

## Cluster configuration

### Infrastructure

First, we need to define the properties for the cluster to works correctly with the Hana configuration.

```
cat <<EOF > /tmp/crm_01.txt
property cib-bootstrap-options: \
        no-quorum-policy=ignore \
        have-watchdog=true \
        stonith-enabled=true \
        stonith-action=reboot \
        stonith-timeout=150s
rsc_defaults rsc-options: \
        resource-stickiness=1000 \
        migration-threshold=5000
op_defaults op-options: \
        timeout=60
EOF
```

And now, load the configuration in the CIB with :

```
crm configure load update /tmp/crm_01.txt
```

### Stonith

Definition of the parameter of SBD

```
cat << EOF > /tmp/crm_02.txt
primitive stonith-sbd stonith:external/sbd \
        params pcmk_delay_max=15 \
        op monitor interval=20 timeout=20
EOF
```

And now, load the configuration in the CIB with :

```
crm configure load update /tmp/crm_02.txt
```

### The ping of the GW

You have to change the IP with the default gateway
```
cat << EOF > /tmp/crm_03.txt
primitive ping_the_gw ocf:pacemaker:ping \
        params host_list=10.117.158.1 multiplier=2000 dampen=60 \
        op start interval=0 timeout=60 \
        op stop interval=0 timeout=20 \
        op monitor interval=10 timeout=60
clone cln_ping_the_gw ping_the_gw \
        meta interleave=true
EOF
```

And now, load the configuration in the CIB with :
```
crm configure load update /tmp/crm_03.txt
```

### Configuration agents for SAPHana and SAPHanaTopology

For SAPHanaTopology

```
cat << EOF > /tmp/crm_04.txt
primitive rsc_SAPHanaTopology_MFZ_HDB00 ocf:suse:SAPHanaTopology \
        operations \$id=rsc_sap2_MFZ_HDB00-operations \
        op monitor interval=10 timeout=600 \
        op start interval=0 timeout=600 \
        op stop interval=0 timeout=300 \
        params SID=MFZ InstanceNumber=00
clone cln_SAPHanaTopology_MFZ_HDB00 rsc_SAPHanaTopology_MFZ_HDB00 \
        meta clone-node-max=1 interleave=true
EOF
```

And now, load the configuration in the CIB with :
```
crm configure load update /tmp/crm_04.txt
```

For SAPHana
```
cat << EOF > /tmp/crm_05.txt
primitive rsc_SAPHana_MFZ_HDB00 ocf:suse:SAPHana \
        operations \$id=rsc_sap_MFZ_HDB00-operations \
        op start interval=0 timeout=3600 \
        op stop interval=0 timeout=3600 \
        op promote interval=0 timeout=3600 \
        op monitor interval=60 role=Master timeout=700 \
        op monitor interval=61 role=Slave timeout=700 \
        params SID=MFZ InstanceNumber=00 PREFER_SITE_TAKEOVER=true DUPLICATE_PRIMARY_TIMEOUT=7200 AUTOMATED_REGISTER=true
ms msl_SAPHana_MFZ_HDB00 rsc_SAPHana_MFZ_HDB00 \
        meta clone-max=2 clone-node-max=1 interleave=true is-managed=true target-role=Master
EOF
```

And now, load the configuration in the CIB with :
```
crm configure load update /tmp/crm_05.txt
```

### Configuration IP for the Hana DB service

```
cat<< EOF > /tmp/crm_06.txt
primitive rsc_ip_MFZ_HDB00 IPaddr2 \
        operations \$id=rsc_ip_MFZ_HDB00-operations \
        op monitor interval=10s timeout=20s \
        params ip=10.117.158.18
EOF
```

And now, load the configuration in the CIB with :
```
crm configure load update /tmp/crm_06.txt
```

### Define Constraints, Colocation and order

```
cat << EOF > /tmp/crm_07.txt
colocation col_saphana_ip_MFZ_HDB00 2000: rsc_ip_MFZ_HDB00:Started msl_SAPHana_MFZ_HDB00:Master
location loc_prefGW_HANA msl_SAPHana_MFZ_HDB00 \
        rule -inf: not_defined pingd or pingd lte 0
order ord_SAPHana_MFZ_HDB00 Optional: cln_SAPHanaTopology_MFZ_HDB00 msl_SAPHana_MFZ_HDB00
EOF
```

And now, load the configuration in the CIB with :
```
crm configure load update /tmp/crm_07.txt
```
