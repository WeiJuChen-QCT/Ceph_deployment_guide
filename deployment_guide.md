# Ceph V17.2.5 QUINCY deploy

Official documents:
https://docs.ceph.com/en/latest/install/

## Requirement
- Python 3 [*Both Server and Clients needed*].
- Systemd.
- Podman or Docker for running containers.
- Time synchronization (such as chrony or NTP).
- LVM2 for provisioning storage devices.

## Test Enviroment

### Software:
|    |  Server  | Client |
| ----- | ---- | ---- |
| OS | Rocky Linux 8.5 (Green Obsidian) | Rocky Linux 8.5 (Green Obsidian) |
| kernel | 4.18.0-348.el8.0.2.x86_64 | 4.18.0-348.el8.0.2.x86_64
| Python | 3.6.8 | 3.6.8 |
| GCC | 8.5.0 20210514 (Red Hat 8.5.0-10) | 8.5.0 20210514 (Red Hat 8.5.0-10) |
| Docker | 20.10.18, build b40c2f6 | N/A |
| Lvm2 | 2.03.14 | N/A |
| Ceph | 17.2.5 quincy | 17.2.5 quincy |


Enable NTP or chrony, Docker.

Disable selinux, firewall.

Requires superuser privileges.


### Tree:
```bash 
Monitors:
 ├──Ceph1: 192.168.0.101 - admin
 ├──Ceph2: 192.168.0.102
 └──Ceph3: 192.168.0.103
Client: 192.168.0.1
```


------------
Installation
------------
### 1. Install cephadm on each nodes:
```
$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
$ chmod +x cephadm
$ ./cephadm add-repo --release quincy
$ ./cephadm install
$ ./cephadm install ceph-common
```
### 2. Create first admin monitor:
```
$ cephadm bootstrap --cluster-network <ip/netmask> --mon-ip <ip>
> cephadm bootstrap --cluster-network 192.168.1.0/24 --mon-ip 192.168.0.101
```
If cluster network is enabled, OSDs will route heartbeat, object replication and recovery traffic over the cluster network. This may improve performance compared to using a single network. We prefer that the cluster network is NOT reachable from the public network or the Internet for added security.

More informations: https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/?highlight=cluster%20network#cluster-network

Output be like:
```
Enabling firewalld port 8443/tcp in current zone...
Ceph Dashboard is now available at:

             URL: https://ceph1:8443/
            User: admin
        Password: xxxxxxx

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/5ef0db52-453a-11ed-a421-54ab3a8a5dba/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/sbin/cephadm shell --fsid 5ef0db52-453a-11ed-a421-54ab3a8a5dba -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/sbin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.
```
Now can check the ceph cluster status.
```
$ ceph -s
```
Output be like:
```
  cluster:
    id:     5ef0db52-453a-11ed-a421-54ab3a8a5dba
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum ceph1-strg35 (age 27m)
    mgr: ceph1-strg35.pumtjb(active, since 25m)
    osd: 0 osds: 0 up (since 19s), 0 in (since 5m)

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

### 3. Add new nodes:
On First admin monitor:
```
$ ssh-copy-id -f -i /etc/ceph/ceph.pub  root@<newnode>
> ssh-copy-id -f -i /etc/ceph/ceph.pub  root@192.168.0.102
> ssh-copy-id -f -i /etc/ceph/ceph.pub  root@192.168.0.103
```
Output be like:
```
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'root@192.168.0.10X'"
```
and check to make sure that only the key(s) you wanted were added.

```
$ ceph orch host add <hostname> <ip>
> ceph orch host add ceph2 192.168.0.102
> ceph orch host add ceph3 192.168.0.103
```
Output:
```
Added host 'ceph2' with addr '192.168.0.102'
Added host 'ceph3' with addr '192.168.0.103'
```
Then check the host list
```
$ ceph orch host ls
```
Output:
```
HOST   ADDR           LABELS  STATUS
ceph1  192.168.0.101  _admin
ceph2  192.168.0.102
ceph3  192.168.0.103
3 hosts in cluster
```
Also, we can add secondary admin host for HA:
```
$ ceph orch host label add ceph2 _admin
```
Then, check host list again, output be like:
```
HOST          ADDR          LABELS  STATUS
ceph1  192.168.0.101  _admin
ceph2  192.168.0.102  _admin
ceph3  192.168.0.103
3 hosts in cluster
```
### 4. Apply OSD:
List all available storage device:
```
$ ceph orch device ls
```
Output be like:
```
HOST   PATH      TYPE  DEVICE ID                           SIZE  AVAILABLE  REFRESHED  REJECT REASONS

ceph1  /dev/sdb  hdd   VBOX_HARDDISK_VB24d7b533-c3ccc5b2  8589M  Yes        23m ago
ceph1  /dev/sdc  hdd   VBOX_HARDDISK_VBb55ca0e6-b341a3b2  8589M  Yes        23m ago
ceph1  /dev/sdd  hdd   VBOX_HARDDISK_VB27942c73-7a238c48  8589M  Yes        23m ago
ceph2  /dev/sdb  hdd   VBOX_HARDDISK_VBd3195806-db05a073  8589M  Yes        23m ago
ceph2  /dev/sdc  hdd   VBOX_HARDDISK_VB9aa9325c-4ab9fa3d  8589M  Yes        23m ago
ceph2  /dev/sdd  hdd   VBOX_HARDDISK_VBb9e601d0-ab6c81e0  8589M  Yes        23m ago
ceph3  /dev/sdb  hdd   VBOX_HARDDISK_VB2f862622-f805e1c0  8589M  Yes        23m ago
ceph3  /dev/sdc  hdd   VBOX_HARDDISK_VB0873d641-b140f7fd  8589M  Yes        23m ago
```
There are two ways to apply osd:

a) Create an OSD from a specific device on a specific host:
```
$ ceph orch daemon add osd <hostname>:<path>
> ceph orch daemon add osd ceph1:/dev/sdb
```
b) Tell Ceph to consume any available and unused storage device:
```
$ ceph orch apply osd --all-available-devices
```
This will automatically add any device to osd when add new device anytime.
To disable this after all device is deploy, add --unmanaged=true 
```
$ ceph orch apply osd --all-available-devices --unmanaged=true
```
After add OSDs, the ```$ ceph orch device ls``` output be like:
```
HOST   PATH      TYPE  DEVICE ID                           SIZE  AVAILABLE  REFRESHED  REJECT REASONS

ceph1  /dev/sdb  hdd   VBOX_HARDDISK_VB24d7b533-c3ccc5b2  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph1  /dev/sdc  hdd   VBOX_HARDDISK_VBb55ca0e6-b341a3b2  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph1  /dev/sdd  hdd   VBOX_HARDDISK_VB27942c73-7a238c48  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph2  /dev/sdb  hdd   VBOX_HARDDISK_VBd3195806-db05a073  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph2  /dev/sdc  hdd   VBOX_HARDDISK_VB9aa9325c-4ab9fa3d  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph2  /dev/sdd  hdd   VBOX_HARDDISK_VBb9e601d0-ab6c81e0  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph3  /dev/sdb  hdd   VBOX_HARDDISK_VB2f862622-f805e1c0  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
ceph3  /dev/sdc  hdd   VBOX_HARDDISK_VB0873d641-b140f7fd  8589M             23m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked
```

### 5. show cluster status:
```
$ ceph -s
```
If everything fine, health part should show HEALTH_OK
Output:
```
  cluster:
    id:     5ef0db52-453a-11ed-a421-54ab3a8a5dba
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph1,ceph2,ceph3 (age 6h)
    mgr: ceph2.nagraj(active, since 30h), standbys: ceph1.fwetao
    osd: 8 osds: 8 up (since 6h), 8 in (since 2d)

  data:
    pools:   2 pools, 1 pgs
    objects: 2 objects, 577 KiB
    usage:   168 MiB used, 64 GiB / 64 GiB avail
    pgs:     1 active+clean
```

Ceph File System Server build
-----------------------------
https://docs.ceph.com/en/quincy/cephfs/createfs/

Ceph file system requires at least 2 pools, 1 data pool and 1 metadata pool:

### 1. create data pool:
```
$ ceph osd pool create <cephfs_data_name>  #replace <cephfs_data_name> with name you want.
> ceph osd pool create cephfs_data
```

### 2. create metadata pool:
```
$ ceph osd pool create <cephfs_metadata_name>  #replace <cephfs_metadata_name> with name you want.
> ceph osd pool create cephfs_metadata
```

### 3. create a file system:
```
$ ceph fs new <cephfs_name> <cephfs_metadata_name> <cephfs_data_name>

#replace <cephfs_metadata_name> and <cephfs_data_name> with same name created in step 1 and 2.
#replace <cephfs_name> with name you want.

> ceph fs new cephfs cephfs_metadata cephfs_data
```
### 4. deploy mds service to the file system above:
```
$ ceph orch apply mds <cephfs_name> --placement="<NUMBER_OF_DAEMONS>
<HOST_NAME_1> <HOST_NAME_2> <HOST_NAME_3>"

#replace <cephfs_name> with same name created in step 3.
#replace <NUMBER_OF_DAEMONS> with amount of mds you want to place(one active and others stanby), then according to <NUMBER_OF_DAEMONS>, replace <HOST_NAME#> with which hostname you want to place the mds services on.

> ceph orch apply mds cephfs --placement="2 ceph1 ceph2"
```
### 5. Configure multiple active mds daemons to scale metadata performance for large systems:
```
$ ceph fs set <cephfs_name> max_mds <NUMBER>

#replace <cephfs_name> with same name created in step 3.
#replcae <NUMBER> with amount of mds you want to active.

> ceph fs set cephfs max_mds 2
```
### 6. check file system status:
```
$ ceph fs status <cephfs_name>  #replace <cephfs_name> with same name created in step 3.
```
Output:
```
cephfs - 0 clients
===
RANK  STATE             MDS            ACTIVITY     DNS    INOS   DIRS   CAPS
 0    active  cephfs.ceph1.ojcope  Reqs:    0 /s    xx      xx     xx      x
     POOL      TYPE     USED  AVAIL
cephfs.meta  metadata  XX.XM  XX.XT
cephfs.data    data     XXXG  XX.XT
         STANDBY MDS
cephfs.ceph2.gceyeb
cephfs.ceph3.ocloem
MDS version: ceph version 17.2.5
```

Mount CephFS client
-------------------

These step execute on client:
### 1. Add Repo to clients
```
$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
$ chmod +x cephadm
$ ./cephadm add-repo --release quincy
```
### 2. Generate a minimal conf file for client host and place it at a standard location:
```
$ mkdir -p -m 755 /etc/ceph
$ ssh <user>@<admin-host> "sudo ceph config generate-minimal-conf" | sudo tee /etc/ceph/ceph.conf
> ssh root@192.168.0.101 "sudo ceph config generate-minimal-conf" | sudo tee /etc/ceph/ceph.conf
$ chmod 644 /etc/ceph/ceph.conf
```
### 3. Create a CephX user and get its secret key:
```
$ ssh <user>@<admin-host> "sudo ceph fs authorize <cephfs_name> client.<name> / rw" | sudo tee /etc/ceph/ceph.client.<username>.keyring

#replace <cephfs_name> with ceph filesystem name created in Ceph File System Server Build step 3.
#replace <name> with cephx username you want.

> ssh root@192.168.0.101 "sudo ceph fs authorize cephfs client.wjchen / rw" | sudo tee /etc/ceph/ceph.client.wjchen.keyring
$ chmod 600 /etc/ceph/ceph.client.<name>.keyring

#replace <name> with cephx username created in step 3.
```
### 4. Mounting CephFS:
There are 2 ways to mount cephfs, through kernel module and by ceph-fuse.
ceph-fuse performance could be relatively lower but FUSE clients can be more manageable, especially while upgrading CephFS.
a) If kernel module:
```
$ ./cephadm install
$ ./cephadm install ceph-common
```
b) If ceph-fuse:
```
$ dnf install ceph-fuse
```
### 5. Mounting CephFS:
```
$ mkdir /mnt/mycephfs
```
a) If using kernel module:
```
$ mount -t ceph <name>@<fsid>.<cephfs_name>=/ /mnt/mycephfs

#replace <name> and <cephfs_name> with same cpehx username and file system name in step 3.
#replace <fsid> with cluster id in /etc/ceph.conf or execute $ ceph -s on ceph admin host.

> mount -t ceph wjchen@58e6f6d4-439a-11ed-8ee7-080027ed8210.cephfs=/ /mnt/mycephfs
```
b) If using ceph-fuse:
```
$ ceph-fuse --id <name> /mnt/mycephfs  #replace <name> with same cephx user name in step 3.
> ceph-fuse --id wjchen /mnt/mycephfs
```
### 6. Unmount:
```
$ umount /mnt/mycephfs
```
### 7. Persistent mount:
Open /etc/fstab and add followings:

a) If using kernel module:
```
<cephuser>@.<cephfs>=/     /mnt/mycephfs    ceph    mon_addr=<ip:6789>,noatime,_netdev    0       0
```
For example:
```
wjchen@.cephfs=/     /mnt/mycephfs    ceph    mon_addr=192.168.0.101:6789,noatime,_netdev    0       0
```


b) If using ceph-fuse:
```
none    /mnt/mycephfs  fuse.ceph ceph.id=<user>[,ceph.conf=<path/to/conf.conf>],_netdev,defaults  0 0
```
For example:
```
none    /mnt/mycephfs  fuse.ceph ceph.id=wjchen,ceph.conf=/etc/ceph/ceph.conf,_netdev,defaults  0 0
```
Then active service through systemctl.
```
$ systemctl start ceph-fuse@/mnt/mycephfs.service
$ systemctl enable ceph-fuse.target
$ systemctl enable ceph-fuse@-mnt-mycephfs.service
```

Block Device
------------
https://docs.ceph.com/en/latest/rbd/rados-rbd-cmds/

### 1. Create a rbd pool:
```
$ ceph osd pool create <rbdpool_name>
$ rbd pool init <rbdpool_name>  #replace <rbdpool_name> with name you want. Do not use "rbd" as <rbdpool_name>.
```

### 2. Create a block device image
```
$ rbd create --size <megabytes> <rbdpool_name>/<image_name>

#replace <megabytes> with size(MB) you want.
#replace <image_name> with any image name.
#replace <rbdpool_name> with same name in step 1.

> rbd create --size 1024 rbd1/image1
```

list rbdpool:
```
$ rbd ls rbd1
```
check rbdpool info:
```
$ rbd info rbd1/image1
```

### 3. Resizing:
```
$ rbd resize --size 2048 <rbdpool_name>/<image_name>  (increase)
$ rbd resize --size 2048 <rbdpool_name>/<image_name> --allow-shrink (decrease)  #replace <rbdpool_name>/<image_name> with the rbdpool/image you want to change size.
```
### 4. Removing:
```
$ rbd rm <rbdpool_name>/<image_name>  #replace <rbdpool_name>/<image_name> with the rbdpool/image you want to remove.
> rbd rm rbd1/image1
```

### 5. Create a block device user(On client):
```
$ ssh root@<admin-host> "ceph auth get-or-create client.<rbd_username> mon 'profile rbd' osd 'profile rbd pool=<rbdpool_name>' mgr 'profile rbd pool=<rbdpool_namae>'" | sudo tee /etc/ceph/ceph.client.<rbd_username>.keyrinig

#replace <rbd_username> with username you want.
#replace <rbdpool_namae> with same name in step 1.

> ssh root@192.168.0.101 "ceph auth get-or-create client.rbdwjchen mon 'profile rbd' osd 'profile rbd pool=rbd1' mgr 'profile rbd pool=rbd1'" | sudo tee /etc/ceph/ceph.client.rbdwjchen.keyring
```

### 6. Map the block device(On client):
```
$ sudo rbd device map <rbdpool_name>/<image_name> --id <rbd_username>

#replace <rbdpool_name>/<image_name> with rbdpool/image you want to mount.
#replace <rbd_username> with username created in step 5.

> sudo rbd device map rbd1/image1 --id rbdwjchen
```
Output be like:
```
/dev/rbd0
```
Then, 
```
$ rbd device list
```
Output:
```
id  pool  namespace  image   snap  device
0   rbd1             image1  -     /dev/rbd0
```

### 7. Unmap block device(On client):
```
$ sudo rbd device unmap /dev/rbd/<rbdpool_name>/<image_name>  #replace <rbdpool_name>/<image_name> with the block device mounted in step 6.
```
or
```
$ sudo rbd device unmap </dev/rbd0>  #replace </dev/rbd0> with the output in step 6.
```

### 8. Mount 
To mount block device, we need to build it to a XFS file system.
```
$ sudo mkfs.xfs </dev/rbd0>  #replace </dev/rbd0> with the output in step 6.
```
Then, make a directory you wnat to mount as endpoint.
```
$ mkdir </Path/to/endpoint>
$ mount </dev/rbd0> </Path/to/endpoint>
```
### 9. Remove Large RBD image:

i). Check large rbd image:
```
$ rbd info rbd1/image1
rbd image 'image1':
        size 1024 TB in 268435456 objects
        order 22 (4096 kB objects)
        snapshot_count: 0
        id: b0071577c29
        block_name_prefix: rbd_data.b0071577c29
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features:
        flags:
        create_timestamp: Fri Nov 18 10:50:24 2022
        access_timestamp: Fri Nov 18 10:50:24 2022
        modify_timestamp: Fri Nov 18 10:50:24 2022
```
ii). Remove header
```
$ rados -p rbd rm rbd_id.image1
$ rados -p rbd rm rbd_header.b0071577c29
```
iii). Remove all rbd data :
```
$ rados -p rbd1 ls | grep '^rbd_data.b0071577c29.' | xargs -n 200  rados -p rbd1 rm
```

### 10. Trobleshooting
i). ```librbd::image::PreRemoveRequest: 0x56544de9d260 check_image_watchers: image has watchers - not removing```

Check rbd/image info first and find prefix name:
```
$ rbd info rbd1/image1
```
Find which device is still watching:
```
$ rados -p rbd1 listwatchers rbd_header.<prefix>
> rados -p rbd1 listwatchers rbd_header.b0071577c29
```
Output be like:
```
watcher=11.50.135.132:0/2286887345 client.4376119 cookie=18446462598732841092
Login to that client and unmap. Then try again.
```


Obeject Gateway
---------------
https://docs.ceph.com/en/latest/cephadm/services/rgw/

### 1. Apply rgw service
a).Let ceph decide which host use as gateway:
```
$ ceph orch apply rgw <rgw_name> #replace <rgw_name> with name you want. Do not use "rgw" as <rgw_name>.
```

b).Designate gateways to specific hosts:
```
$ ceph orch host label add <rgw_host1> <rgw-name>
$ ceph orch host label add <rgw_host2> <rgw-name>
$ ceph orch apply rgw <rgw-name> '--placement=label:rgw count-per-host:2' --port=8000

#replace <rgw_host#> with hostname you want to place the rgw service.
#replace <rgw_name> with name you want. Do not use "rgw" as <rgw_name>.
```



### 2. Create a user (S3 interface):
https://docs.ceph.com/en/latest/radosgw/admin/
```
$ radosgw-admin user create --uid=<username> --display-name="<display-name>" [--email=<email>](email is optional)

#replace <username> with username you want.
#replace <display-name> with username you want to display.

> radosgw-admin user create --uid=wjchen --display-name="WJ Chen" --email=wjchen@example.com
```
### 3. Create a subuser (Swift interface): 
Must specify the user ID (--uid={username}), a subuser ID and the access level for the subuser.
```
$ radosgw-admin subuser create --uid={uid} --subuser={uid} --access=[ read | write | readwrite | full ]
> radosgw-admin subuser create --uid=wjchen --subuser=wjchen:swift --access=full
```
### 4. check user info:
```
$ radosgw-admin user info --uid=<username>  #replace <username> with uid created in step 2.
```

### 5. Modify user info:
```
$ radosgw-admin user modify --uid=<username> --display-name="<display-name>"
$ radosgw-admin subuser modify --uid=<username> --subuser=<username>:swift --access=full
```


### 6. Set limitation:
```
$ radosgw-admin quota set --quota-scope=user --uid=<username> --max-objects=1024 --max-size=1024
```
then enable:
```
$ radosgw-admin quota enable/disable --quota-scope=user --uid=<username>
```
### 7. User Enable/Suspend:
When you create a user, the user is enabled by default. However, you may suspend user privileges and re-enable them at a later time. To suspend a user, specify user suspend and the user ID.
```
$ radosgw-admin user suspend --uid=<username>
```
To re-enable a suspended user, specify user enable and the user ID:
```
$ radosgw-admin user enable --uid=<username>
```


Power off all cluster SOP
-------------------------
1. Ensure no client mount fs, block or gateway.
2. Ensure all PGs are active+clean by checking ceph -s from an admin node.
3. From an admin node, set the following flags:
```
$ ceph osd set noout
$ ceph osd set norecover 
$ ceph osd set norebalance
$ ceph osd set nobackfill
$ ceph osd set nodown
$ ceph osd set pause
```
4. If on their own dedicated nodes, power off the MDS and RGW nodes.
5. Shutdown OSD nodes one by one.
6. Shutdown monitor nodes one by one.
7. Shutdown admin node.


Power On cluster SOP
--------------------------
If network equipment was involved, ensure it is powered on and stable prior to powering on any Ceph hosts/nodes
1. Power on the admin node
2. Power on the monitor nodes
3. Power on the OSD nodes
4. Wait for all the nodes to come up , Verify all the services are up and the connectivity is fine between the nodes.
5. If on their own nodes, power on the MDS and RGW nodes.
6. From an admin node, unset the following flags:
```
$ ceph osd unset noout
$ ceph osd unset norecover
$ ceph osd unset norebalance
$ ceph osd unset nobackfill
$ ceph osd unset nodown
$ ceph osd unset pause
```
7. Check and verify the cluster is in healthy state, Verify all the clients are able to access the cluster.



Configure Pool's pg
--------------------------
```
                (OSDs * 100)
   Total PGs =  ------------
                 pool size
```
The result should be rounded up to the nearest power of two.
For example ,for a cluster with 200 OSDs and a pool size of 3 replicas, you would estimate your number of PGs as follows:
```
   (200 * 100)
   ----------- = 6667. Nearest power of 2: 8192
        3
```
Recommendation is 100 to 200 PGs per OSD.

The Ceph Storage Cluster has a default maximum value of 300 placement groups per OSD. 

//////NOT CONFIRM ///////
/////You can set a different maximum value in your Ceph configuration file.
/////$ ceph config set global mon_max_pg_per_osd <NUMBER>
/////////////////////////

```
$ ceph osd pool set <pool-name> pg_autoscale_mode <mode>
$ ceph config set global osd_pool_default_pg_autoscale_mode <mode>
```

```<pool-name>``` is pool you want to change. execute ```$ ceph df```, or ```$ ceph osd lspools```, to check pool list.
  
```<mode>``` should be on of on|off|warn
- off: Disable autoscaling for this pool. It is up to the administrator to choose an appropriate pgp_num for each pool. Please refer to Choosing the number of Placement Groups for more #information.
- on: Enable automated adjustments of the PG count for the given pool.
- warn: Raise health alerts when the PG count should be adjusted

Then, we can change the pg number:
```
$ ceph osd pool set <pool-name> pg_num <number>
> ceph osd pool set fs1.data pg_num 512
```
If you don’t specify the number of placement groups, Ceph will use the default value of 32
In order to avoid health warning POOL_PG_NUM_NOT_POWER_OF_TWO messages, use values which are power of two for the placement groups.


Configure mtu
--------------------------

MTU on each node should be same otherwise heart beat between osds will failed then the ceph server will down. 



Delete pool
--------------------------
By default, pool cannot be delete. To delete a pool, execute following command:
$ ceph config set mon mon_allow_pool_delete true

Then, delete a pool
$ ceph osd pool delete <pool_name> <pool_name> --yes-i-really-really-mean-it
#replace <pool_name> with pool you want to delete and repeat twice.


Cache Tiering
--------------------------
https://docs.ceph.com/en/latest/rados/operations/cache-tiering

### 1. Prepare OSDs for cache.
List current crush class and move osds to another class for cache. If osds are already has diffirent class(ssd or nvme), these steps can be skip. 
```
$ ceph osd crush class ls
$ ceph osd crush rm-device-class osd.<osd_id>
$ ceph osd crush set-device-class <class> osd.<osd_id>
#<class> is one of hdd, ssd and nvme.
#Execute $ceph osd tree to find osd id.
> ceph osd crush set-device-class nvme 1 2 3
```
### 2. Create a crush rule for cache:
```
$ ceph osd crush rule create-replicated <rule_name> default host <crush_class>
> ceph osd crush rule create-replicated nvme_rule default host nvme
$ ceph osd crush rule ls
```

### 3. Create a pool for cache rule:
```
$ ceph osd pool create <cache_pool> <rule_name>
#replace <rule_name> with name in step 2.
> ceph osd pool create cache1 nvme_rule
```

### 4. Setting up caching
Setting a cache tier involves associating a backing storage pool with a cache pool
```
$ ceph osd tier add <storage_pool> <cache_pool>
> ceph osd tier add fs1.data cache1
```
To set the cache mode, execute the following:
```
$ ceph osd tier cache-mode <cache_pool> <mode>
#mode is one of writeback| readproxy| readonly| none
> ceph osd tier cache-mode cache1 writeback
```

The cache tiers overlay the backing storage tier, so they require one additional step: 
you must direct all client traffic from the storage pool to the cache pool.
To direct client traffic directly to the cache pool, execute the following:
```
$ ceph osd tier set-overlay <storage_pool> <cache_pool>
> ceph osd tier set-overlay fs1.data cache1
```
### 5. Configure parameters
Ceph’s production cache tiers use a Bloom Filter:
```
$ ceph osd pool set <cache_pool> hit_set_type bloom
> ceph osd pool set cache1 hit_set_type bloom
```

The hit_set_count and hit_set_period define how many such HitSets to store, and how much time each HitSet should cover.
```
$ ceph osd pool set <cache_pool> hit_set_count <count>
$ ceph osd pool set <cache_pool> hit_set_period <period(s)> 
> ceph osd pool set <cache_pool> hit_set_count 12
> ceph osd pool set <cache_pool> hit_set_period 14400 #14400 s
#A larger hit_set_count results in more RAM consumed by the ceph-osd process.
```

The cache tiering agent performs two main functions:
Flushing: The agent identifies modified (or dirty) objects and forwards them to the storage pool for long-term storage.
Evicting: The agent identifies objects that haven’t been modified (or clean) and evicts the least recently used among them from the cache.

Absolute Sizing:
```
$ ceph osd pool set <cache_pool> target_max_bytes <bytes>
$ ceph osd pool set <cache_pool> target_max_objects <objects>
> ceph osd pool set cache1 target_max_bytes 1099511627776  #1TB
> ceph osd pool set cache1 target_max_objects 1000000  #1M objects
```

Relative Sizing:
```
$ ceph osd pool set <cache_pool> cache_target_dirty_ratio <ratio>
$ ceph osd pool set <cache_pool> cache_target_dirty_high_ratio <ratio>
$ ceph osd pool set <cache_pool> cache_target_full_ratio <ratio>
> ceph osd pool set cache1 cache_target_dirty_ratio 0.4  #40%
> ceph osd pool set cache1 cache_target_dirty_high_ratio 0.6  #60%
> ceph osd pool set cache1 cache_target_full_ratio 0.8  #80%
```
Cache age:
```
$ ceph osd pool set cache1 cache_min_flush_age <time(s)>
$ ceph osd pool set cache1 cache_min_evict_age <time(s)>
> ceph osd pool set cache1 cache_min_flush_age 600  #600s
> ceph osd pool set cache1 cache_min_evict_age 1800  #1800s
```

### 6. Remove cache tiering:
Remove objects in cache pool
```
$ rados -p <cache_pool> cache-flush-evict-all
```
Change cache mode to readproxy then none:
```
$ ceph osd tier cache-mode <cache_pool> readproxy
$ ceph osd tier cache-mode <cache_pool> none
```
Remove overlay:
```
$ ceph osd tier remove-overlay <storage_pool>
```
Remove cache tiring:
```
$ ceph osd tier remove <storage_pool> <cache_pool>
```
### 7. Additional autorization for RBD cache tiering:
```
$ ssh root@<admin-host> "ceph auth get-or-create client.<rbd_username> mon 'profile rbd' osd 'profile rbd pool=<rbdpool_name>' osd 'allow rwx pool=<rbd_cache_pool>' mgr 'profile rbd pool=<rbdpool_name>'" | sudo tee /etc/ceph/ceph.client.<rbd_username>.keyring

> ssh ceph1 "ceph auth get-or-create client.rbdwjchen mon 'profile rbd' osd 'profile rbd pool=rbd1' osd 'allow rwx pool=cache2' mgr 'profile rbd pool=rbd1'" | sudo tee /etc/ceph/ceph.client.rbdwjchen.keyring
```
