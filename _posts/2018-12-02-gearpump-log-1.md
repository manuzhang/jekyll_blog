--- 
comments: true
layout: post
title: Gearpump Log #1
---

``` 
[INFO] Gearpump retires from Apache Incubator 
```

After incubating for two and a half years at Apache Software Foundation (ASF), Gearpump recently [retired from Apache incubator](http://mail-archives.apache.org/mod_mbox/incubator-general/201809.mbox/%3CCABT57mYLRAYvVS5N2GrD-07ddAdgkf42ibZmdXpeGQHAf%2BwcDg%40mail.gmail.com%3E) and [returned to GitHub home](https://github.com/gearpump/gearpump). 

> Hi all,

> Gearpump has been an Apache incubator since 2016-03-08. Thank you all for help to incubate Gearpump and keep it move forward.

> The activity around Gearpump has, however, slowed down and the last release was more than one year ago. There is no sign of a growing community, which is a requirement to be a Apache TLP. The long period of release process has made things even harder for such a slim community.

> I don't see a future for Gearpump within Apache so it's better off to leave.

> What do you think ?

> Thanks,   
> Manu Zhang 

After had been the only one to develop Gearpump for a while, I finally started the discussion to retire it on dev list. We reached consensus and passed a formal vote. After another vote passed on general list, Gearpump officially retired. **Note retirement doesn't mean the end of this project and Gearpump will continue on GitHub reserving all commits.** The [Apache repo](https://github.com/apache/incubator-gearpump) is archived. 

```
[INFO] Gearpump continues on GitHub
```

A new journey has started. From now on, I'd like to log everything (commits, issues, thoughts, etc) of Gearpump . I regret not having done so earlier as I can hardly remember the early design decisions we made and why. 

The first thing I did is adding [scala-steward](https://github.com/fthomas/scala-steward) integration which helps to keep Gearpump's dependencies up-to-date. Scala-steward robot has opened 55 pull requests and I've merged 5 of them to update Akka, algebird, etc. It will periodically check for updates.

Then I planed to decompose the huge codebase into various projects under [gearpump group](https://github.com/gearpump) since it would be too much for me to maintain and upgrade dependencies. For example, the `external-kafka` module depends on Kafka 0.8.2.1 which has no scala 2.12 release. I drew a graph of [inter-project dependency](https://github.com/gearpump/gearpump/issues/2089#issuecomment-439678535) with [sbt-project-graph](https://github.com/dwijnand/sbt-project-graph) and ported the external modules to [gearpump-externals](https://github.com/gearpump/gearpump-externals) along with corresponding examples. Nonetheless, I quitted mid-way when I found it's too hard to separate out the remaining modules. Given everything is already archived at the Apache repo, I simply [removed non-core modules](https://github.com/gearpump/gearpump/commit/42b13c09b5e3192b0a3b760594e4ebe5b67bd68a). 


As Gearpump is no longer an Apache project, the package must be renamed from `org.apache.gearpump` and ASF license headers removed. It's a [big change](https://github.com/gearpump/gearpump/commit/25bb3e04a5feb6e4fe639a5eea78893822f365a7) and renaming package with Intellij pushed my 2015 Mac Pro to the limit. A [follow-up fix](https://github.com/gearpump/gearpump/commit/7c94cfdb7673e9b05cd836c983c3e2fddf1d59ad) had to be made for what I had left out. 

Upgrading to Scala 2.12 had been on my schedule for a while but was blocked by an ancient version of [Upickle](https://github.com/lihaoyi/upickle). Upickle is the json serialization library for communication between Gearpump's dashboard and backend. The usage of Upickle has changed a lot. It used to generate reader/writers for a case class automatically while they have to be [defined manually now](https://github.com/gearpump/gearpump/commit/6c9727df81089d78627dd00de7219db56b0dbcdb#diff-315a1559a02aa4b74e0c56c13c345485). 

[Upgrading to Scala 2.12](https://github.com/gearpump/gearpump/commit/78bed0b457ce7e7c4cd57f5ad6502a880f5f66f4) itself was not too much work. From the [release blog](https://www.scala-lang.org/news/2.12.0/) of Scala 2.12.0

> Although Scala 2.11 and 2.12 are mostly source compatible to facilitate cross-building, they are not binary compatible

Some notable differences are 

1. These `scalacOption`s, `-Yinline-warnings`, `-Yclosure-elim` and `-Yinline`, no longer exist. Here are some [recommended flags for 2.12](https://tpolecat.github.io/2017/04/25/scalac-flags.html).
2. Type inference. `-Xsource:2.11` can get you the old behavior and tell whether the failure is brought by changes in 2.12. 
    
    a. 
    
    ``` scala
    trait BasicService {
      private val LOG: Logger = LogUtil.getLogger(getClass) 
    }
    // [error] overloaded method value getLogger with alternatives
    // [T](clazz: Class[T]org.slf4j.Logger <and>
    // [T](clazz: Class[T], context: String ...)org.slf4j.Logger
    
    private val LOG: Logger = LogUtil.getLogger(classOf[BasicService]) //OK
    ```
    
    b.
    
    ```scala
    class DummyRunner[T] extends FlatMapper[T, T](FlatMapFunction(Option(_)), "")
    // [error] missing parameter type for expanded function
    
    class DummyRunner[T] extends FlatMapper[T,T](
        FlatMapFunction((t => Option(t)): T => TraversableOnce[T]), "") //OK
    
    ```


With these major changes, I [bumped up the target version of next release](https://github.com/gearpump/gearpump/commit/08e2ac4762aa9a73ea424f1b19484e209663c9aa) from `0.8.5-SNAPSHOT` to `0.9.0-SNAPSHOT`. Next step would be more testing and make sure there is no regression with [Beam's Gearpump runner](https://github.com/apache/beam/tree/master/runners/gearpump). If everything is OK, then `0.9.0` is good to go.

```
[INFO] 13 commits, 8 issues opened and 6 resolved after the return of Gearpump
```






 





