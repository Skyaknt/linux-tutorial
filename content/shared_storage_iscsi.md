## Shared storage on the network with iSCSI
Many ways to share storage on a network exist. The iSCSI protocol defines a way to see a remote blocks device as a local disk. A remote device on the network is called iSCSI Target, a client which connects to iSCSI Target is called iSCSI Initiator.

### iSCSI Target Setup
Install admin tools first, configure target to persistantly start at boot time and then start it
```
yum -y install targetcli
systemctl enable target
systemctl start target
```
To start using ``targetcli``, run it and to get a layout of the tree interface, run ls
```
targetcli
targetcli shell version 2.1.fb37
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.
/> 
...
```
#### Create a Backstore
Backstores enable support for different methods of storing an object on the local machine. Creating a storage object defines the resources the backstore will use. The supported backstores are: block devices, files, pscsi and ramdisks. Block device ``/dev/sdc`` in our case.
```
/> /backstores/block create name=block_storage dev=/dev/sdc
Created block storage object block_backend using /dev/sdc.
```
#### Create an iSCSI Target
Create an iSCSI target using a specified name
```
/> iscsi/ create iqn.2017-10.com.noverit.centos:3260
Created target iqn.2015-05.com.noverit.caldara02:3260.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260
```
#### Configure an iSCSI Portal
An iSCSI Portal is an object specifying the IP address and port where the iSCSI target listen to incoming connections.
```
/> /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/portals/ create
Using default IP port 3260
Binding to INADDR_ANY (0.0.0.0)
This NetworkPortal already exists in configFS
```
By default, a portal is created when the iSCSI Target is created listening on all IP addresses (0.0.0.0) and the default iSCSI port 3260. Make sure that the 3260 is not used by another application, else specify a different port.

#### Configure the LUNs
A Logical Unit Number (LUN) is a number used to identify a logical unit, which is a device addressed by the standard SCSI protocol or Storage Area Network protocols which encapsulate SCSI, such as Fibre Channel or iSCSI itself.  To configure LUNs, create LUNs of already created storage objects.
```
/> /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/luns/ create /backstores/block/block_storage
Created LUN 0.
```

#### Configure Access List
Create an Access List for each initiator (iscsi client) that will be connecting to the target (iscsi server). This enforces authentication when that initiator connects, allowing only LUNs to be exposed to each initiator. Usually each initator has exclusive access to a LUN. All initiators have unique identifying names IQN. The initiator's unique name IQN must be known to configure ACLs.

Check the iSCSI initiator IQN on the initiator machine you want to give access. For example, for open-iscsi initiator on linux systems, the IQN can be found in the ``/etc/iscsi/initiatorname.iscsi`` file. 

Back to the target machine and configure Access List to permit access from the initiator machine
```
/> cd /iscsi/iqn.2017-10.com.noverit.centos:3260/tpg1/
/> /acls create iqn.1994-05.com.redhat:client00.noverit.com
Created Node ACL for iqn.1994-05.com.redhat:client00.noverit.com
Created mapped LUN 0.
/> cd iqn.1994-05.com.redhat:client00.noverit.com
/> set auth userid=user
Parameter userid is now 'user'.
/> set auth password=password
Parameter password is now 'password'.
```

Repeat the steps above for each initiator you need to give permission.

At the end of configuration, the iSCSI target envinronment should look like the following
```
/> ls
o- / .............................................................[...]
  o- backstores ..................................................[...]
  | o- block .....................................................[Storage Objects: 1]
  | | o- block_storage ...........................................[/dev/sdc (12.0GiB) write-thru activated]
  | |   o- alua ..................................................[ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................[ALUA state: Active/optimized]
  | o- fileio ....................................................[Storage Objects: 0]
  | o- pscsi .....................................................[Storage Objects: 0]
  | o- ramdisk ...................................................[Storage Objects: 0]
  o- iscsi .......................................................[Targets: 1]
  | o- iqn.2017-10.com.noverit.centos:3260 .......................[TPGs: 1]
  |   o- tpg1 ....................................................[no-gen-acls, no-auth]
  |     o- acls ..................................................[ACLs: 4]
  |     | o- iqn.1994-05.com.redhat:client00.noverit.com .........[Mapped LUNs: 1]
  |     | | o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     | o- iqn.1994-05.com.redhat:kubew03 ......................[Mapped LUNs: 1]
  |     | | o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     | o- iqn.1994-05.com.redhat:kubew04 ......................[Mapped LUNs: 1]
  |     | | o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     | o- iqn.1994-05.com.redhat:kubew05 ......................[Mapped LUNs: 1]
  |     |   o- mapped_lun0 .......................................[lun0 block/block_storage (rw)]
  |     o- luns ..................................................[LUNs: 1]
  |     | o- lun0 ................................................[block/block_storage (/dev/sdc)]
  |     o- portals ...............................................[Portals: 1]
  |       o- 0.0.0.0:3260 ........................................[OK]
  o- loopback ....................................................[Targets: 0]

Save and exit

/> saveconfig
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
```

The ``/etc/target/saveconfig.json`` file contains the above configuration.

Restart the target service 
```
systemctl restart target
```
### iSCSI Initiator Setup
After configuring the iSCSI on the target machine, move to setup the iSCSI initiator machine and install the iSCSI initiator

```
yum -y install iscsi-initiator-utils
```

Check the IQN
```
cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1994-05.com.redhat:2268c31791
```

The last field of the IQN above is the identifier of the the above initiator machine. Change it to a more meaningful string, for example the host name.

```
echo "InitiatorName=iqn.1994-05.com.redhat:client00.noverit.com" > /etc/iscsi/initiatorname.iscsi
```

Edit the ``/etc/iscsi/iscsid.conf`` initiator configuration file
```bash
# line 54: uncomment
node.session.auth.authmethod = CHAP
# line 58,59: uncomment and specify the username and password you set on the iSCSI target server
node.session.auth.username = user
node.session.auth.password = password
```

The iSCSI initiator is composed by two services, iscsi and iscsid, start both and enable to start at system startup
```
systemctl start iscsid iscsi
systemctl enable iscsid iscsi
systemctl status iscsid iscsi
```

To connect the target, first discover the published iSCSI resouces and then login
```
iscsiadm --mode discovery --type sendtargets --portal centos:3260 --discover
10.10.10.2:3260,1 iqn.2017-10.com.noverit.centos:3260
```

confirm status after discovery
```
iscsiadm -m node -o show 
```

Login to the target
```
iscsiadm --mode node --targetname iqn.2017-10.com.noverit.centos:3260 --portal caldara02:3260 --login
```

Check the storage block devices on the initiator machine
```
lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                         8:0    0 232.9G  0 disk
├─sda1                      8:1    0   500M  0 part /boot
├─sda2                      8:2    0  73.4G  0 part
│ ├─os-swap               253:0    0   3.9G  0 lvm  [SWAP]
│ ├─os-root               253:1    0    50G  0 lvm  /
│ └─os-data               253:2    0 178.5G  0 lvm  /data
└─sda3                      8:3    0   159G  0 part
  └─os-data               253:2    0 178.5G  0 lvm  /data
sdc                         8:32   0    20G  0 disk
```
The disk ``/dev/sdc`` is the remote iSCSI block device exported by the target. It is seen as local block device in the initiator machine. The disk can be used as with standard local disk commands and configurations, including ``fdisk``, ``mkfs``, ``e2label``, etc.


To disconnect the remote devices
```
iscsiadm --mode node --targetname iqn.2017-10.com.noverit.centos:3260 --portal caldara02:3260  --logout
```

Stop and then disable the services at startup, if required
```
systemctl stop iscsid iscsi
systemctl disnable iscsid iscsi
```


