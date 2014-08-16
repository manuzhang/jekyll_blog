---
author: manuzhang
comments: true
date: 2012-07-14 02:17:05+00:00
layout: post
slug: greenplum-load-from-file
title: greenplum load from file
wordpress_id: 183
categories:
- database
- linux
- 实习笔记
tags:
- greenplum
---

上篇笔记讲了 greenplum 的安装， 这篇讲如何将文件中的数据导入 greenplum database。



这些文件 （part-000*, 20多个） 都在一个目录 [cci]/home/gpadmin/external[/cci] 下， 文件有两列， 用 tab 隔开， 至少有几十万行。 我要建一张 friendlist 表， 它有两个属性 uid 和 friendid， 都是字符串类型， 和文件中的那两列对应。 大致上是两类方法， 一类是建一个 external table， 另一类就是真的将数据导入到数据库中。 最终尝试成功的是第二类方法。



<!-- more -->



# I. External table



先拓宽一下自己的知识。





<blockquote>
  External tables allow you to access external files **as though** they were regular database tables


</blockquote>



首先， 既然是 “as though”， 那就没有在数据库中建表。 Oracle 的[文档](http://docs.oracle.com/cd/B10500_01/server.920/a96652/ch11.htm)里讲它不支持 DML 操作， 也不能建索引。 它的作用是作为一个向数据库中真正的表中导入数据的数据源， 它和直接导入数据相比的好处参看链接， 因为没有真正体验过， 现在还讲不出所以然来。 greenplum 提供了 readable 和 writable 两种类型的 external table， 并且还支持读入网络文件中的数据。  需要指出的是， writable 只允许 insert 操作， 写好的数据是不能修改（删除）的。 讲具体操作。 这里用到了 gpfdist （greenplum parallel file server）， 它是一个服务器， 之后创建的 external table 就从它那里“索取”数据。



[cce]
gpfdist -p 8086 -d /home/gpadmin/external -l /home/gpadmin/gpAdminlogs &
[/cce]
[cci]-p[/cci] 参数指定了端口号， [cci]-d[/cci] 参数指定外部文件所在的文件夹， [cci]-l[/cci] 参数指定日志文件的位置。 下面在 greenplum 里操作。


[cc lang="sql"]
create external table friendlist ( uid text, friendid text)
location ('gpfdist://mdw:8086/home/gpadmin/external/*')
format 'text' (delimiter E't');
[/cc]

external table 支持 text 和 csv 两种文件格式， 不是 csv 的就都设为 text 吧。 delimiter 就是在文件中讲列隔开的标记， 注意转移字符的写法。 可惜失败了， 提示是 404 错误， 找不到文件。



下面讲第二类方法。 当然先建表再说。



[cc lang="sql"]
create table friendlist (uid text, friendid text);
[/cc]

按照文档的说法， 用 char, varchar 和 text 的查询效率没差。 导入也有两种方法， 一种是用 copy 语句， 另一种是在外部用 gpload。





# II. COPY



[cc lang="sql"]
copy friendlist from '/home/gpadmin/external/part-00000' with delimiter E't';
[/cc]

比较讨厌的是 copy 不支持 wildcard， 我有二十多个文件啊， 还有 copy 下一个文件的数据时会把上个文件的覆盖吗？ （后经验证， 不会） 我能想到的是写个脚本合并文件。



[cce]
# /bin/sh -
for file in 'ls /home/gpadmin/external'
do
   cat /home/gpadmin/external/$file >> part-all
done
[/cce]

刚开始把脚本文件和数据文件放在一起， 把脚本文件的内容都加进去了。 可以统计一下行数， 看是否正确。


[cc]
find . -name "/home/gpadmin/external/*" | xargs wc -l
wc -l part-all
[/cc]

如果不加 xargs， 就是统计文件的个数。 这样就可以像上面那条命令一样把 part-all 中的数据导入， 就不再重复了。





# III. gpload



用 gpload， 要写一个 yaml 控制文件, my_load.yml。 在 yaml 里， 缩进和空格都是要命的 （花费了我大把青春）。


[cc lang="yaml"]
 ---
 VERSION: 1.0.0.1
 DATABASE： template1
 USER: gpadmin
 HOST: mdw
 GPLOAD:
 INPUT:
    - SOURCE:
        LOCAL_HOSTNAME:
            - mdw
        PORT: 8086
        FILE:
            - /home/gpadmin/external/* 
    - COLUMNS:
        - UID: text
        - FRIENDID: text
    - FORMAT: text
    - DELIMITER: E't'
 OUTPUT:
    - TABLE: friendlist
    - MODE: insert
[/cc]

接着 gpload 调用控制文件导入数据。



[cc]
gpload -f my_load.yml
[/cc]

提示找不到 friendlist， 怎么会呢？



[cc lang="sql"]
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'; 
[/cc]

(参考 [search for tables in Postgres](http://www.linuxscrew.com/2009/07/03/postgresql-show-tables-show-databases-show-columns/))

明明是有 friendlist 的， 困顿了半天， 试了下



[cc lang="sql"]
create table public.friendlist (uid text, friendid text);
[/cc]

结果行了。 gpload 事实上是调用了 gpfdist 先建一个 external table， 之后再通过 insert, update 或 merge 操作将数据导入到目标表中。 因此它拥有 gpfdist 的 parallel 特性。 与之相比， copy 就是由一个进程完成， 它没有 external table 的代价。 没有比较过， 感觉上 gpload 会快。 当然， 如果只需要做查询操作的话， 那么只建 external table 的代价是最小的。 最后再提醒一下， 使用 gpfdist， gpload 之前都需要先



[cc]
source $GPHOME/greenplum_path.sh
[/cc]

