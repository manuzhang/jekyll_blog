---
comments: true
layout: post
title: greenplum 数据备份与恢复
---

前面介绍了 greenplum 的安装和从文件中加载数据， 下面讲讲 greenplum 数据备份与恢复 （有关的文件夹都延续前面的设置）。

## I. 备份

由于 community edition 只支持一个结点， 我们考虑在多台机器上安装 greenplum （参考前面的文章）， 先将数据加载到一台机器的一个数据库中， 再将这个数据库复制到其它机器上。 greenplum 提供了数据备份和恢复机制来实现数据的转移。

![greenplum_backup](https://lh4.googleusercontent.com/-Cv3XgIsYk3A/UAoSzDs_s8I/AAAAAAAAAZ8/G_JbEgRk0Bw/s853/greenplum_backup.png)


备份的过程非常简单， 就是 master host 和 segment host 分别把自己的数据复制出来， 由统一的时间戳标识， 恢复就是逆过程。


我们在 192.168.1.101 上有一个数据库 test_db， 现在把它复制到 192.168.1.103 （它们在同一个局域网内， 且两台机器的 greenplum 配置相同） 上。

第一步是在 192.168.1.101 上备份。

```bash
source $GPHOME/greenplum_path.sh
gp_dump test_db
```

如果没有报错的话， 会在 masterdata/gpseg-1 下生成如下格式的文件

```
gp_cdatabase_1_<dbid>_<timestamp>
gp_dump_1_<dbid>_<timestamp>
gp_dump_1_<dbid>_<timestamp>_post_data
gp_dump_status_1_<dbid>_<timestamp>
```

官方指南里说还有一个

```
gp_catlog_1_<dbid>_<timestamp>
```

没有发现， 对于后面的恢复也没有影响。 把数据复制过去。

```bash
scp root@192.168.1.101:/home/gpadmin/masterdata/gp_cdatabase*  
root@192.168.1.103：/home/gpadmin/masterdata
scp root@192.168.1.101:/home/gpadmin/masterdata/gp_dump*  
root@192.168.1.103:/home/gpadmin/masterdata
```

在 `segmentdata/gpseg0` 和 `segmentdata/gpseg1` 下生成如下格式的文件

```
gp_dump_0_<dbid>_<timestamp>
gp_dump_status_0_<dbid>_<timstamp>
```

同样复制过去

```bash
scp root@192.168.1.101:/home/gpadmin/segmentdata/gpseg0/gp_dump* 
root@192.168.1.103:/home/gpadmin/segmentdata/gpseg0 
scp root@192.168.1.101:/home/gpadmin/segmentdata/gpseg1/gp_dump* 
root@192.168.1.103:/home/gpadmin/segmentdata/gpseg1
```

好了， 登录到 192.168.1.103 上。 先新建一个 test_db 数据库 (这边就省略 source 的步骤了）

```bash
createdb test_db
```



## II. 恢复

```
gp_restore --gp-k=<timestamp> -d test_db
```

这里的 timestamp 就是生成的备份文件中的统一时间按戳。 结果失败了。 查阅一下原因

```bash
cat masterdata/gpseg-1/gp_restore_status_1_<dbid>_<timestamp>
```

有一行是

```
/home/gpadmin/masterdata/gpseg-1/./gp_dump_1_<dbid>_<timestamp>: Permission denied
```

看来是没有读备份文件的权限， 查一下果然只有 root 有读写权限， 想起前面复制的时候都是以 root 的身份完成的， 也就可以理解了。 将这些文件的 owner 和 group 改为 gpadmin 之后， 就可以正常恢复了。

大约 200G 的数据， 备份三个多小时， 恢复八个多小时。



