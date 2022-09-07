| 编号  | 系统         | 角色               | IP             | Ceph版本 |
| --- | ---------- | ---------------- | -------------- | ------ |
| 1   | Centos 8.5 | 引导节点，mon，mgr，osd | 192.168.30.200 | Quincy |
| 2   | Centos 8.5 | mon，mgr，osd,rgw  | 192.168.30.201 | Quincy |
| 3   | Centos 8.5 | mon，mgr          | 192.168.30.202 | Quincy |

作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：[xiaohui_li@foxmail.com](mailto:xiaohui_li@foxmail.com)

# 先决条件准备

除非另有说明，不然先决条件需在每个参与节点准备

1. Python 3
2. Systemd
3. Podman 3
4. Chrony
5. LVM2

```bash
yum install podman chrony lvm2 systemd python3 -y
reboot
```

为ceph准备仓库

```ini
cat > /etc/yum.repos.d/cephadm.repo << eof
[cephadm]
name=cephadm lixiaohui write
baseurl=http://mirrors.aliyun.com/centos/8-stream/storage/x86_64/ceph-quincy
enabled=1
gpgcheck=0
eof
```

安装cephadm

```bash
yum install cephadm -y
```

# 引导新集群

本操作将创建 Ceph 集群的第一个"监控守护程序”，后期Ceph 会随着集群的增长自动部署监控守护程序，Ceph 会在集群收缩时自动缩减监控守护进程。此自动增长和收缩的顺利执行取决于正确的子网配置。如果集群中的所有 ceph 监控守护程序都位于同一子网中，则无需手动管理 ceph 监控守护程序。 将根据需要自动向子网添加多个监视器，因为新主机已添加到群集

要将 Ceph 集群配置为在单个主机上运行，请在引导时使用--single-host-defaults标志

```bash
cephadm bootstrap --mon-ip 192.168.30.200 --allow-fqdn-hostname
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman (/usr/bin/podman) version 3.3.1 is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
Cluster fsid: a13e967e-07fb-11ed-920e-000c29759605
Verifying IP 192.168.30.200 port 3300 ...
Verifying IP 192.168.30.200 port 6789 ...
Mon IP `192.168.30.200` is in CIDR network `192.168.30.0/24`
Mon IP `192.168.30.200` is in CIDR network `192.168.30.0/24`
Internal network (--cluster-network) has not been provided, OSD replication will default to the public_network
Pulling container image quay.io/ceph/ceph:v17...
Ceph version: ceph version 17.2.1 (ec95624474b1871a821a912b8c3af68f8f8e7aa1) quincy (stable)
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
firewalld ready
Enabling firewalld service ceph-mon in current zone...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting mon public_network to 192.168.30.0/24
Wrote config to /etc/ceph/ceph.conf
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Creating mgr...
Verifying port 9283 ...
firewalld ready
Enabling firewalld service ceph in current zone...
firewalld ready
Enabling firewalld port 9283/tcp in current zone...
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/15)...
mgr not available, waiting (2/15)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for mgr epoch 5...
mgr epoch 5 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to /etc/ceph/ceph.pub
Adding key to root@localhost authorized_keys...
Adding host host1.xiaohui.cn...
Deploying mon service with default placement...
Deploying mgr service with default placement...
Deploying crash service with default placement...
Deploying prometheus service with default placement...
Deploying grafana service with default placement...
Deploying node-exporter service with default placement...
Deploying alertmanager service with default placement...
Enabling the dashboard module...
Waiting for the mgr to restart...
Waiting for mgr epoch 9...
mgr epoch 9 is available
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
firewalld ready
Enabling firewalld port 8443/tcp in current zone...
Ceph Dashboard is now available at:

             URL: https://sarize.gicscorple.com:8443/
            User: admin
        Password: larb16bhvr

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/a13e967e-07fb-11ed-920e-000c29759605/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/sbin/cephadm shell --fsid a13e967e-07fb-11ed-920e-000c29759605 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/sbin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.
```

上述命令将：

1. 为本地主机上的新群集创建监视器和管理器守护程序。

2. 为 Ceph 集群生成新的 SSH 密钥，并将其添加到root用户的文件: /root/.ssh/authorized_keys

3. 将公钥的副本写入: /etc/ceph/ceph.pub

4. 将最小配置文件写入/etc/ceph/ceph.conf，需要此文件才能与新群集进行通信。

5. 将管理（特权 client.admin） 密钥的副本写入: /etc/ceph/ceph.client.admin.keyring

6. 将标签添加到引导主机。默认情况下，任何具有此标签的主机都将（也）获得 和 的副本。_admin, /etc/ceph/ceph.conf, /etc/ceph/ceph.client.admin.keyring

目前我们需要继续完成集群，下图要求我们需要先扩展集群，因为我们只有一个mon节点，没有osd等其他节点，无法开始工作

<img src="file:///C:/Users/Xiaohui/AppData/Roaming/marktext/images/2022-07-20-15-23-13-image.png" title="" alt="" width="450">

<img src="file:///C:/Users/Xiaohui/AppData/Roaming/marktext/images/2022-07-20-15-29-06-image.png" title="" alt="" width="452">

```bash
[root@host1 ~]# cephadm shell -- ceph -s
Inferring fsid a13e967e-07fb-11ed-920e-000c29759605
Inferring config /var/lib/ceph/a13e967e-07fb-11ed-920e-000c29759605/mon.host1.xiaohui.cn/config
Using ceph image with id 'e5af760fa1c1' and tag 'v17' created on 2022-06-23 19:49:45 +0000 UTC
quay.io/ceph/ceph@sha256:d3f3e1b59a304a280a3a81641ca730982da141dad41e942631e4c5d88711a66b
  cluster:
    id:     a13e967e-07fb-11ed-920e-000c29759605
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum host1.xiaohui.cn (age 16m)
    mgr: host1.xiaohui.cn.oslofw(active, since 14m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

# 添加存储节点

显示在集群中所有的存储设备

```bash
[ceph: root@host1 /]# ceph orch device ls
HOST              PATH          TYPE  DEVICE ID                                   SIZE  AVAILABLE  REFRESHED  REJECT REASONS
host1.xiaohui.cn  /dev/nvme0n2  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
host1.xiaohui.cn  /dev/nvme0n3  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
host1.xiaohui.cn  /dev/nvme0n4  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
```

添加单独的设备到集群

```bash
[ceph: root@host1 /]# ceph orch daemon add osd host1.xiaohui.cn:/dev/nvme0n2
Created osd(s) 0 on host 'host1.xiaohui.cn'
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.19530  root default
-3         0.19530      host host1
 0    ssd  0.19530          osd.0       up   1.00000  1.00000
[ceph: root@host1 /]# ceph osd ls
0
```

将集群中所有未使用的存储设备添加进来

```bash
[ceph: root@host1 /]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.19530  root default
-3         0.19530      host host1
 0    ssd  0.19530          osd.0       up   1.00000  1.00000
 1               0  osd.1             down   1.00000  1.00000
 2               0  osd.2             down   1.00000  1.00000
[ceph: root@host1 /]# ceph osd ls
0
1
2
```

如需删除某个osd，可使用以下方法

```bash
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.58589  root default
-3         0.58589      host host1
 0    ssd  0.19530          osd.0       up   1.00000  1.00000
 1    ssd  0.19530          osd.1       up   1.00000  1.00000
 2    ssd  0.19530          osd.2       up   1.00000  1.00000
[ceph: root@host1 /]# ceph orch osd rm 0
Scheduled OSD(s) for removal
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.39059  root default
-3         0.39059      host host1
 1    ssd  0.19530          osd.1       up   1.00000  1.00000
 2    ssd  0.19530          osd.2       up   1.00000  1.00000
```

删除之后需要清理磁盘，使其再次可用

```bash
[ceph: root@host1 /]ceph-volume inventory /dev/nvme0n2

====== Device report /dev/nvme0n2 ======

     path                      /dev/nvme0n2
     ceph device               None
     lsm data                  {}
     available                 False
     rejected reasons          Insufficient space (<10 extents) on vgs, locked, LVM detected
     device id                 VMware_Virtual_NVMe_Disk_VMware_NVME_0000
     removable                 0
     ro                        0
     vendor
     model                     VMware Virtual NVMe Disk
     sas address
     rotational                0
     scheduler mode            none
     human readable size       200.00 GB
    --- Logical Volume ---
     name                      osd-block-3fb41ef8-517b-4297-94c0-9c19074ad938
     osd id                    0
     cluster name              ceph
     type                      block
     osd fsid                  3fb41ef8-517b-4297-94c0-9c19074ad938
     cluster fsid              a13e967e-07fb-11ed-920e-000c29759605
     osdspec affinity          None
     block uuid                oZa15c-o9Ip-20Dj-7lkK-UkC6-vZzc-09M8Qr
[ceph: root@host1 /]# lvremove /dev/ceph-9d69f228-af91-4c7a-b124-d8a603de5549/osd-block-3fb41ef8-517b-4297-94c0-9c19074ad938
File descriptor 4 (/dev/tty) leaked on lvremove invocation. Parent PID 8: bash
Do you really want to remove active logical volume ceph-9d69f228-af91-4c7a-b124-d8a603de5549/osd-block-3fb41ef8-517b-4297-94c0-9c19074ad938? [y/n]: y
  Logical volume "osd-block-3fb41ef8-517b-4297-94c0-9c19074ad938" successfully removed.

[ceph: root@host1 /]# vgremove ceph-9d69f228-af91-4c7a-b124-d8a603de5549
File descriptor 4 (/dev/tty) leaked on vgremove invocation. Parent PID 8: bash
  Volume group "ceph-9d69f228-af91-4c7a-b124-d8a603de5549" successfully removed
[ceph: root@host1 /]# pvremove /dev/nvme0n2
File descriptor 4 (/dev/tty) leaked on pvremove invocation. Parent PID 8: bash
  Labels on physical volume "/dev/nvme0n2" successfully wiped.

[ceph: root@host1 /]# wipefs -a /dev/nvme0n2
[root@host1 ~]# systemctl stop ceph-a13e967e-07fb-11ed-920e-000c29759605@osd.0.service
```

# 添加主机到集群

需要注意，新主机必须满足此文章的先决条件

在新主机的root用户文件中安装集群的公有 SSH 密钥

```bash
[root@host1 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@host2
[root@host1 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@host3
```

更新集群主机列表

在此过程中，将会自动扩展mon和mgr节点，以及自动应用其未使用的osd设备，可使用ceph orch host rm删除主机，--offline --force参数可以用于已经脱机的删除场景，可以用--labels=my_label1,my_label2的方式添加标签

```bash
[ceph: root@host1 /]# ceph orch host add host2.xiaohui.cn 192.168.30.201
Added host 'host2.xiaohui.cn' with addr '192.168.30.201'
[ceph: root@host1 /]# ceph orch host add host3.xiaohui.cn 192.168.30.202
Added host 'host3.xiaohui.cn' with addr '192.168.30.202'
[ceph: root@host1 /]# ceph orch host ls
HOST              ADDR            LABELS  STATUS
host1.xiaohui.cn  192.168.30.200  _admin
host2.xiaohui.cn  192.168.30.201
host3.xiaohui.cn  192.168.30.202
3 hosts in cluster
```

将服务分配到指定主机

```bash
[ceph: root@host1 /]# ceph orch apply rgw host2.xiaohui.cn
Scheduled rgw.host2.xiaohui.cn update...
[ceph: root@host1 /]# ceph orch ps
...
rgw.host2.xiaohui.cn.host1.ldnxhc  host1.xiaohui.cn  *:80         running (104s)    95s ago  104s    20.2M        -  17.2.1     e5af760fa1c1  7a1aabfa738b
rgw.host2.xiaohui.cn.host2.azjzas  host2.xiaohui.cn  *:80         running (101s)    96s ago  101s    17.2M        -  17.2.1     e5af760fa1c1  06f755f1c03b
```
