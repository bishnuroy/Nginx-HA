<h2 align="center"> Nginx-HA-Setup </h2>



<>We will do this configuration on 2 nodes </b>

  - <b>Node1:</b> nginx-node1.example.com - 192.168.1.10
  - <b>Node2:</b> nginx-node2.example.com - 192.168.1.11

  - <b>Floation IP:</b> nginxlb.example.com- 192.168.1.100
  - <b>Tools:</b> Nginx, DRBD, Crosync, Pacemaker and crm command line.
  
    - <b>DRBD:</b>
    
      DRBD(Distributed Replicated Block Device) using this for shared volume.
      DRBD reference document link is https://docs.linbit.com/
      
    - <b>Corosync & Pacemaker:</b>
    
      Corosync is an open source program that provides cluster membership and messaging capabilities, often referred to as the       messaging layer, to client servers. 
      
      Pacemaker is an open source cluster resource manager (CRM), a system that coordinates resources and services that are         managed and made highly available by a cluster. In essence, Corosync enables servers to communicate as a cluster, while       Pacemaker provides the ability to control how the cluster behaves.
    
    - <b>Ref Document: </b>
      
      https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/
      
      http://crmsh.github.io/man/
      
      https://www.howtoforge.com/tutorial/how-to-set-up-nginx-high-availability-with-pacemaker-corosync-and-crmsh-on-ubuntu-1604/
      
      https://www.suse.com/documentation/sle_ha/book_sleha/data/sec_ha_config_crm.html
      
      https://geekpeek.net/linux-cluster-resources/
     

<h2 align="center"> DRBD SETUP </h2>


<b>Step1:</b>

  - Add separate 5GB volume in each nodes.
  - Create vg(vg_nginx) and lvm(lv_nginx) with this volume on both the nodes.
```
Create volume group.

# vgcreate vg_nginx /dev/sdb

Create lvm with the volume group

## lvcreate -n lv_nginx -l 90%VG /dev/vg_nginx
```
<b>Step2:</b> 
 - Install drbd service in both the nodes.
```
#rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm -y
#rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org

Check the pakage info
# yum info *drbd* | grep Name

#yum install drbd90-utils drbd90-utils-sysvinit drbdlinks kmod-drbd90 -y

Check the drbd module installed or not 
# lsmod | grep -i drbd 

If not reboot the system and execute below ommands.
# find / -name drbd.ko

Get the proper module and isert it.
# insmod /usr/lib/modules/3.10.0-957.1.3.el7.x86_64/weak-updates/drbd90/drbd.ko

Now execute below command you will get following outputs.
# lsmod | grep -i drbd
       drbd                  553913  0
       libcrc32c              12644  2 xfs,drbd
```
<b>Step3</b>
 
 - Configure DRBD
 - Add file "/etc/drbd.d/ngr401.res" on both the node with below containt.
 
```console
resource ngr01 {
protocol C;
	on nginx-node1 {
                device /dev/drbd0;
                disk /dev/vg_nginx/lv_nginx;
                address 192.168.1.10:7788;
                meta-disk internal;
        }
        on nginx-node1 {
                device /dev/drbd0;
                disk /dev/vg_nginx/lv_nginx;
                address 192.168.1.11:7788;
                meta-disk internal;
        }
}
```
 - Now execute below command on both the node.
```
 # drbdadm create-md ngr01
 
    --==  Thank you for participating in the global usage survey  ==--
The server's response is:
you are the 3036th user to install this version
initializing activity log
initializing bitmap (144 KB) to all zero
Writing meta data...
New drbd meta data block successfully created.
```
```
# lsblk

NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0                   2:0    1    4K  0 disk
sda                   8:0    0   32G  0 disk
├─sda1                8:1    0    1G  0 part /boot
└─sda2                8:2    0   31G  0 part
  ├─rootvg-system   253:0    0    5G  0 lvm  /
  ├─rootvg-opt      253:1    0    5G  0 lvm  /opt
  └─rootvg-var      253:2    0    2G  0 lvm  /var
sdb                   8:16   0    5G  0 disk
└─vg_nginx-lv_nginx 253:3    0  4.5G  0 lvm
sr0                  11:0    1 1024M  0 rom
```
 - execute below command on master node.
```
#drbdadm primary ngr01
#drbdadm -- --overwrite-data-of-peer primary ngr01/0
```
```
# cat /proc/drbd
version: 9.0.16-1 (api:2/proto:86-114)
GIT-hash: ab9777dfeaf9d619acc9a5201bfcae8103e9529c build by mockbuild@, 2018-11-03 13:54:24
Transports (api:16): tcp (9.0.16-1)
```
```
#drbdadm status
ngr01 role:Primary
  disk:UpToDate
  nginx-node1 role:Secondary
    peer-disk:UpToDate
```
Execute Below command on both the nodes
```
# mkfs.xfs /dev/drbd0

meta-data=/dev/drbd0             isize=512    agcount=4, agsize=294645 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1178579, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
<b>Step4</b>
 - Create a drive where DRBD drive will mounted.
 ```
 # mkdir /nginx
 ```
 
 <h2 align="center"> Crosync, Pacemaker & CRM Setup </h2>
 
 <b>Step1:</b>
  - Install the repo and packages.
 ```
#wget https://download.opensuse.org/repositories/network:ha-clustering:Stable/CentOS_CentOS-7/network:ha-clustering:Stable.repo

#cp network\:ha-clustering\:Stable.repo /etc/yum.repos.d/crmsh.repo

#yum install crmsh -y
#yum install pacemaker corosync -y
#systemctl enable corosync
#systemctl enable pacemaker

 ```
 <b>Step2:</b>
  - Create file "/etc/corosync/corosync.conf" with below contents on both the nodes.
 ```
 totem {
	version: 2
	secauth: off
	cluster_name: nginx_lb
	transport: udpu
	rrp_mode: passive
}

nodelist {
	node {
      		ring0_addr: 192.168.1.10
          name: tom
      		nodeid: 1
	}
	node {
    		ring0_addr: 192.168.1.11
        name: jerry
    		nodeid: 2
	}
}

quorum {
	provider: corosync_votequorum
	two_node: 1
	wait_for_all: 1
	last_man_standing: 1
	auto_tie_breaker: 0
}

logging {
        to_syslog: yes
        logfile: /var/log/cluster/corosync.log
}
 ```
 
 <b>Step3:</b>
  - Restart The service now.
 ```
 # systemctl restart corosync pacemaker
 
 # systemctl status corosync pacemaker
 
 Check the status
 
 # crm status
 
 Stack: corosync
Current DC: bishnu (version 1.1.19-8.el7_6.2-c3c624ea3d) - partition with quorum
Last updated: Wed Jan  2 11:42:42 2019
Last change: Wed Jan  2 11:34:52 2019 by hacluster via crmd on tom

2 nodes configured
0 resources configured

Online: [ tom jerry ]

No resources
 ```
<b>Step4:</b>

```
# crm configure show

node 1: tom
node 2: jerry
property cib-bootstrap-options: \
	have-watchdog=false \
	dc-version=1.1.19-8.el7_6.2-c3c624ea3d \
	cluster-infrastructure=corosync \
	cluster-name=nginx_lb \
	stonith-enabled=false \
	no-quorum-policy=ignore
	
```
 
<h3 align="center"> Input resource in corosync:</h3>

<b>Step1:</b>
 - Configure heartbeat: virtip_res resource
 ```
 # crm configure
crm(live)configure# primitive virtip_res ocf:heartbeat:IPaddr2 \
   > params ip="192.168.1.100" cidr_netmask="24" \
   > op start interval="0" timeout="100s" \
   > op stop interval="0" timeout="100s" \
   > op monitor interval="10s" timeout="100s"
crm(live)configure# commit
 ```
<b>Step2:</b>

 - DRBD resource define:
```
crm(live)configure# primitive drbd_res ocf:linbit:drbd \
   params drbd_resource=ngr01 \
   op monitor interval="29" role="Master" \
   op monitor interval="30" role="Slave" \
   op start interval="0" timeout="240s" \
   op stop interval="0" timeout="100s"
crm(live)configure# commit
```
 - drbd master slave:
 
```
crm(live)configure# ms drbd_ms drbd_res \
    meta master-max="1" master-node-max="1" clone-max="2" \
    clone-node-max="1" notify="true"
crm(live)configure# commit
```
<b>Step3:</b>

 - DRBD FileSystem:
 
 ```
 crm(live)configure# primitive fsdrbd_res ocf:heartbeat:Filesystem \
   > params device="/dev/drbd0" directory="/nginx" fstype="xfs" \
   > options="noatime" \
   > op start interval="0" timeout="100s" \
   > op stop interval="0" timeout="100s" \
   > op monitor interval="10s" timeout="100s"
crm(live)configure# commit
```
<b>Step4:</b>

 - Create Nginx service resource.
 
```
crm(live)configure# primitive nginx_res systemd:nginx \
   > op start interval="0" timeout="100s" \
   > op stop interval="0" timeout="100s" \
   > op monitor interval="20s" timeout="100s"
crm(live)configure# commit
```
<b>Step5:</b>
 - Enable resource
```
Create Group:

[root@nginx-node1 ~]# crm configure
crm(live)configure# group grp_ngipfs virtip_res fsdrbd_res
crm(live)configure# group grp_nginx nginx_res
crm(live)configure# commit
```
Create_Collection:
```
crm(live)configure# colocation col_ng_with_drbd \
   > inf: grp_ngipfs nginx_res drbd_ms:Master
crm(live)configure# commit
```
Create Order:

```
crm(live)configure# order ord_ngnix \
   > inf: drbd_ms:promote grp_ngipfs:start grp_nginx:start
crm(live)configure# commit
```

 <b>Step6:</b>
 
 - Enabled Enginx config folder
    - Make sure services are running in master node.
    - Install nginx in active node.
    - Now follow the following instractions.
```
#mv /etc/nginx/conf.d /nginx

#ln -s /nginx/conf.d /etc/nginx/

#Reboot the server
```
- After reboot the service will moved to 2nd node.
- Install nginx in 2nd node.
- Delete folder "/etc/nginx/conf.d"
- Execute following instraction on 2nd active node.
```
#rm -rf /etc/nginx/conf.d

#ln -s /nginx/conf.d /etc/nginx/
```
- Check the node1 is it up or down.
- If Node1 is up reboot the node2.
- All the the servies will move to Node1.
 
Now execute following command on master node.
```
crm resource start virtip_res 
crm resource start fsdrbd_res
crm resource start nginx_res
```
 - Now again reboot node1 once its up reboot node2.
 
 <b>Now you are ready to use your nginx</b>
 
 <h2 align="center"> Enjoy Now </h2>

