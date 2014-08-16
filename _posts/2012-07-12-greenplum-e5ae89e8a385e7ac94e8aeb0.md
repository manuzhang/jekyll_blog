---
comments: true
layout: post
title: greenplum 安装笔记
---

这两天接到任务，在实验室的机器上安装 greenplum database，这里记录一下安装过程，备忘。主要参考官方的安装指南， google 一下也有很多安装笔记。

## I. 安装环境

greenplum community edition 的系统要求是 RHEL 5.0 以上或是 OS X， 我觉得是 unix/linux 系统的都可以试试。 文件系统推荐 xfs， ext3 的话支持 [root filesystem](http://www.linfo.org/root_filesystem.html)， 我最终在 ext4 上尝试， 也行。 最低内存，16 GB RAM per server，这个我的机子就吃不消了。 刚开始找的机器是 RHEL 4.0，一直不知情， 直到在运行 greenplum bin 目录下的一些程序时报 “libc.so.6 version 'GLIBC_2.4' not found" 的错时才发现。 一个系统版本一个 glibc 版本 （除非自己编译）， 要升级 glibc， 就要升级系统。 查看 RHEL 版本:

```bash
cat /etc/redhat-release
```

果然如此。 换一台机器， 这次先检查了版本, 是 RHEL 6.0。

## II. 系统配置

命令默认是以 root 用户执行。
greenplum 对系统参数有要求， 主要是共享内存， 网络和用户数限制三部分。 Linux 上是修改 /etc/sysctl.conf 和 /etc/security/limits.conf 两个文件。

```
 # /etc/sysctl.conf

 # xfs_mount_options = rw,noatime,inode64,allocsize=16m
 kernel.shmmax = 500000000 
 kernel.shmmni = 4096 
 kernel.shmall = 4000000000 
 kernel.sem = 250 512000 100 2048 
 kernel.sysrq = 1 
 kernel.core_uses_pid = 1 
 kernel.msgmnb = 65536 
 kernel.msgmax = 65536 
 kernel.msgmni = 2048 
 net.ipv4.tcp_syncookies = 1 
 net.ipv4.ip_forward = 0 
 net.ipv4.conf.default.accept_source_route = 0 
 net.ipv4.tcp_tw_recycle = 1 
 net.ipv4.tcp_max_syn_backlog = 4096 
 net.ipv4.conf.all.arp_filter = 1 
 net.ipv4.ip_local_port_range = 1025 65535 
 net.core.netdev_max_backlog = 10000 
 vm.overcommit_memory = 2 
```

由于文件系统不是 xfs, 注释了第一行。

```
 # /etc/security/limits.conf

 *   soft nofile 65536
 *   hard nofile 65536
 *   soft nproc 131072
 *   hard nproc 131072 
```

使新的配置生效。

```bash
 sysctl -p 
```

开始时不知道要做这一步， 后面初始化系统时 greenplum 就会抱怨共享内存不足。 还要为磁盘设置 [I/O scheduler](http://en.wikipedia.org/wiki/I/O_scheduling)， 及 [read-ahead](http://www.overclock.net/t/388475/linux-performance-tuning) 值。

```bash
 echo deadline > /sys/block/devname/queue/scheduler 
 blockdev --getra /dev/devname 
 blockdev --setra 16385 /dev/devname 
```

这里的 devname 包括所有的磁盘 （sda，sdb， ...）。 最后在 [cci]/etc/hosts[/cci] 里加入 greenplum 系统中所有的机子。 由于只支持一个结点， 加入本机作为 master host， 还有一个 segment host。 （参见 greenplum 的[系统架构](https://lh3.googleusercontent.com/-r-6ZtddyLEA/UBYxMM1IJuI/AAAAAAAAAak/UNgijBV1lFY/w367-h276-n-k/Screenshot%2Bfrom%2B2012-07-30%2B15%253A00%253A25.png)）

```bash
# /etc/hosts

 127.0.0.1 mdw 
 127.0.0.1 sdw 
```

不要直接用 localhost， 初始化时 greenplum 会抱怨的。

## III. 正式安装

下面解压 greenplum 安装包， 运行安装文件。

```bash
 unzip greenplum-db-4.2.x.x-*PLATFORM*.zip 
 ./greenplum-db-4.2.x.x-*PLATFORM*.bin 
```

一路 yes 或 ENTER (如果没有先前的版本需要迁移）， 默认安装在 `/usr/local/greenplum-db` （可以加到环境变量 $GPHOME 里）。

## IV. 添加管理员 （system user）

新建一个 all_hosts 文件， 加入 master host, segment host， 一行一个。

```
  mdw 
  sdw 
```

下面给 greenplum 加管理员 （system user）。

```bash
  source /usr/local/greenplum-db/greenplum_path.sh 
  gpseginstall -f all_hosts -u gpadmin -p *PASSWORD* 
```

greenplum_path.sh 里是 greenplum 的环境变量， 如果在运行 $GPHOME/bin 下的程序时出错， 很可能是没有先做 source 那一步。 这样系统里就多了一个 gpadmin 用户， /home 下会多一个 gpadmin 文件夹。 gpadmin 里应该会有一个 gpAdminlogs 文件夹， 里面是日志文件， 出错时极有用。

在 gpadmin 里新建两个数据文件夹， masterdata，  segmentdata， 将它们的 owner 改为 gpadmin。 官方指南里还要配置 NTP 同步时间， 这里单个结点， 就跳过这一步了。

以上的命令都是 root 下执行的， 现在加了 gpadmin， 登入到 gpadmin， 加一下 $GPHOME， 在 gpadmin 加入和之前一样的 all_hosts， 验证之前的设置是否正确。

```bash
 source $GPHOME/greenplum_path.sh 
 gpcheck -f all_hosts -m mdw 
```

还可以验证 Disk I/O 和 Memory Bandwidth。 新建一个 seg_hosts， 里面是 segment hosts （不要加 master host)

```bash
 source $GPHOME/greenplum_path.sh 
 gpcheckperf -f seg_hosts -r ds -D -d segmentdata 
```

注意对 segmentdata 得有读写权限 （这里出错似乎可以不用管）。

## V. 初始化系统

下面初始化整个系统。 先创建配置文件， 可以修改 greenplum 提供的例子。

```bash
 cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config 
 /home/gpadmin/gpconfigs/gpinitsystem_config 
```

修改如下：

```bash
 ARRAY_NAME="EMC Greenplum DW" 
 SEG_PREFIX=gpseg 
 PORT_BASE=40000 
 declare -a DATA_DIRECTORY=(/home/gpadmin/segmentdata /home/gpadmin/segmentdata) 
 MASTER_HOSTNAME=mdw 
 MASTER_DIRECTORY=/home/gpadmin/masterdata 
 MASTER_PORT=5432 
 TRUSTED SHELL=ssh 
 CHECK_POINT_SEGMENT=8 
 ENCODING=UNICODE 
```

至少得有两个 segment 文件夹。 在主机间建立信任关系：

```bash
 gpssh-exkeys -f /home/gpadmin/all_hosts
```

下面正式开始：

```bash
 gpinitsystem -c gpconfigs/gpinitsystem_config -h gpconfigs/seg_hosts 
```

比较悲剧的是文档没看仔细， 我用了 `all_hosts`， 不知道会有什么恶果。 最后设置一下 `MASTER_DATA_DIRECTORY` 环境变量。
在 `~/.bash_profile` 里加

```bash
 export MASTER_DATA_DIRECTORY=/home/gpadmin/masterdata/gpseg-1 
```

然后 `source ～/.bash_profile`。

这样算是装完了。 试一下， 数据库应该已经启动了 （没有的话， `gpstart`）。

```sql
 psql postgres 
 create database test_db (id integer, name text); 
 insert into test_db values (1, 'manu'); 
 insert into test_db values (2, 'zhang'); 
 select * from test_db; 
 drop database test_db; 
```

退出。

```
 \q
```

## 参考链接

  1. [单机安装Greenplum的小结](http://blog.csdn.net/yqlong000/article/details/7476745)
  2. [greenplum 安装与初始化单机版](http://blog.csdn.net/moxpeter/article/details/7287222)


