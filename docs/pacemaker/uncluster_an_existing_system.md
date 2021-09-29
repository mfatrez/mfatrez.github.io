---
title: "Uncluster Pacemaker Cluster"
date: 2021-09-23
draft: true
---

# Scenario

* node01 is primary
* node02 is secondary

# Part 1

As root user
```
node01:~ # systemctl disable pacemaker
rm '/etc/systemd/system/multi-user.target.wants/pacemaker.service'
node02:~ # systemctl disable pacemaker
rm '/etc/systemd/system/multi-user.target.wants/pacemaker.service'
```

```
node01:~ # rcpacemaker stop
node02:~ # rcpacemaker stop
```

As mfzadm

```
mfzadm@node01:/usr/sap/MFZ/HDB00> hdbnsutil -sr_unregister --name=MFZ_node1
unregistering site ...
nameserver node01:30001 not responding.
checking for inactive nameserver ...
nameserver node01:30001 not responding.
nameserver is shut down, proceeding ...
nameserver node01:30001 not responding.
nameserver node01:30001 not responding.
nameserver node01:30001 not responding.
nameserver node01:30001 not responding.
nameserver node01:30001 not responding.
nameserver node01:30001 not responding.
Opening persistence ...
run as transaction master
updating topology for system replication takeover ...
mapped host node02 to node01
sending unregister request to primary site (1) ...
warning: unregistration request to primary failed, please run   hdbnsutil -sr_unregister --id=1   on primary site

#####################################################################################
### CAUTION: You must start the database in order to complete the unregistration! ###
#####################################################################################

done.
```

```
mfzadm@node01:/usr/sap/MFZ/HDB00> HDB start


StartService
Impromptu CCC initialization by 'rscpCInit'.
  See SAP note 1266393.
OK
OK
Starting instance using: /usr/sap/MFZ/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function StartWait 2700 2


17.01.2018 17:50:37
Start
OK

17.01.2018 17:52:05
StartWait
OK
```

As root user
```
node01:~ # ip addr add 10.117.158.11/32 dev eth0
```

# Part 2

As mfzadm user

```
mfzadm@node02:/usr/sap/MFZ/HDB00> hdbnsutil -sr_cleanup --force
cleaning up ...

###################################################################################################################################
### WARNING: cleaning up will break system replication; secondary sites need to be takeovered by issuing hdbnsutil -sr_takeover ###
###################################################################################################################################

checking for inactive nameserver ...
nameserver node02:30001 not responding.
clearing local ini files ...
Opening persistence ...
run as transaction master
clearing topology ...
done.
```

```
mfzadm@node02:/usr/sap/MFZ/HDB00> HDB start


StartService
Impromptu CCC initialization by 'rscpCInit'.
  See SAP note 1266393.
OK
OK
Starting instance using: /usr/sap/MFZ/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function StartWait 2700 2


17.01.2018 17:50:54
Start
OK
```
