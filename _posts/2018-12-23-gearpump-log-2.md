--- 
comments: true
layout: post
title: Gearpump Log &#35;2
---

Each pull request on Gearpump will be checked for [code coverage](https://codecov.io/gh/gearpump/gearpump) thanks to [sbt-scoverage](https://github.com/scoverage/sbt-scoverage) plugin. `sbt coverage test` had been added to `.travis.yml` to trigger the check.

The coverage suddenly dropped to 0 when I [upgraded Gearpump's Scala version to 2.12](https://github.com/gearpump/gearpump/pull/2106). The [suspicious change](https://github.com/gearpump/gearpump/commit/78bed0b457ce7e7c4cd57f5ad6502a880f5f66f4#diff-a84f91f20f040218bccd09fed4761fb3) was upgrading sbt-scoverage from `1.2.0` to `1.5.1`. I thought the problem was related to the sbt version `0.13.16` and [asked about it on sbt-scoverage's issues](https://github.com/scoverage/sbt-scoverage/issues/270). Meanwhile, I went on to try my luck with sbt `1.2.7` which turned a minor issue into significant work because all SBT plugins had to be upgraded to versions that work with SBT `1.x` 

One roadblock was we'd maintained forked versions of [sbt-assembly](https://github.com/sbt/sbt-assembly) and [sbt-pack](https://github.com/xerial/sbt-pack) from `0.13.x` and it looked very hard to port those changes. 

### Upgrading sbt-assembly
Gearpump uses sbt-assembly to create fat jars with shaded dependencies for sub-modules. Tinkering with sbt-assembly has contributed to my blog series *Shade with SBT [I](https://manuzhang.github.io/2016/10/15/shading.html), [II](https://manuzhang.github.io/2016/11/12/shading-2.html) and [III](https://manuzhang.github.io/2017/04/21/shading-3.html)*. There was one more issue. [Running gearpump examples in Intellij or SBT would fail](https://issues.apache.org/jira/browse/GEARPUMP-162) with `ClassNotFoundException` since they had **provided** dependency on `gearpump-core` and `gearpump-streaming`. My solution then was changing the scope to **compile** and [manually filtering out everything other than examples' own source codes](https://github.com/apache/incubator-gearpump/pull/200/files#diff-7f7e51fd877cefd5501e7567e56a9858R180) with the built-in `assemblyExcludedJars` option. There was one limitation with the option that it only excluded jars. Hence, I went on to make a tweak [extending the exclusion to all files](https://github.com/manuzhang/sbt-assembly/commit/344e092171c3e6f29f73b73df58884b239573915#diff-c670e50888639e8441bc5754a62689dfR182). 

Apparently, it's not worth maintaining a tweak and porting it from one forked version to another. Therefore, I [switched to the official sbt-assembly 0.14.9](https://github.com/gearpump/gearpump/commit/f6497e9d470dc8fbc70edb44cc68db1bb54ccdc8#diff-a84f91f20f040218bccd09fed4761fb3R3) and used the `assemblyMergeStrategy` option this time.

```scala
assemblyMergeStrategy in assembly := {
x =>
  // core and streaming dependencies are not marked as provided
  // such that the examples can be run with sbt or Intellij
  // so they have to be excluded manually here
  if (x.contains("examples")) {
    val oldStrategy = (assemblyMergeStrategy in assembly).value
    oldStrategy(x)
  } else {
    MergeStrategy.discard
  }
}
```

It's working! [At least 50%](https://github.com/gearpump/gearpump/issues/2118). 

This brought in another issue that [the assembled jar were not published as before](https://github.com/gearpump/gearpump/issues/2120) with the following.

```
addArtifact(Artifact("gearpump-core"), sbtassembly.AssemblyKeys.assembly)
```

After taking a closer look, I found the previous way was fragile where the unassembled and assembled jars were created with the same name in different directories and the latter **happened** to override the former when publishing. The [fix](https://github.com/gearpump/gearpump/pull/2122/files) was to append an `assembly` to the assembled jar and alter the artifact setting [as documented](https://github.com/sbt/sbt-assembly#publishing-not-recommended).

```scala
artifact in (Compile, assembly) := {
  val art = (artifact in (Compile, assembly)).value
  art.withClassifier(Some("assembly"))
}
```

By the way, it's not recommended to publish fat jars so I think this merits revisiting later.

### Upgrading sbt-pack

`sbt-pack` has served to create distributable Gearpump packages with launch scripts under `bin` and system jars (output from sbt-assembly) under `lib`. The `lib` directory is by default put on all classpaths in the launch scripts. Not to conflict with users' dependencies, Gearpump tried to put system dependencies under subdirectories of `lib`. My colleague Vincent [hacked into the packLibDir option](https://github.com/huafengw/sbt-pack/commit/88fd39e01a263e142c8668124beb88018b58ef69) and made the following possible.

```
packLibDir := Map(	      
  "lib/hadoop" -> new ProjectsToPack(gearpumpHadoop.id).
      exclude(services.id, core.id),
  "lib/services" -> new ProjectsToPack(services.id).exclude(core.id)
)
```

Like sbt-assembly, I decided not to maintain the forked sbt-pack given the cost of porting. [The solution was to pack system dependencies into other directories](https://github.com/gearpump/gearpump/commit/d2655ed5b2629ae7bb31f19caf595c2b46683588) on the same level as `lib` and I was able to upgrade sbt-pack to 0.11.

### Other changes

* [sbt-unidoc has been upgraded to 0.4.2](https://github.com/gearpump/gearpump/commit/801101732e315a63ba02af9d926f83df65cf5fb0) with [unresolved Javadoc errors](https://github.com/gearpump/gearpump/issues/2117).

* [License headers have been updated removing ASF declarations](https://github.com/gearpump/gearpump/commit/801101732e315a63ba02af9d926f83df65cf5fb0) with the help of [Intellij's updating copyright text button](https://www.jetbrains.com/help/idea/copyright.html).

* Build scripts have been moved from classes extending `sbt.Build` in various `.scala` files to `build.sbt`.  

### Is the coverage issue solved ?

No! Unfortunately the coverage issue was not solved even when [SBT had been upgraded 1.2.7](https://github.com/gearpump/gearpump/commit/331f6e0f76dce8145a9fef286e82ef210861375b). It's turned out that SBT version is not the scapegoat but that I have not read sbt-scoverage's README carefully enough. `sbt coverage` only outputs the coverage data and `sbt coverageReport` is needed to generate the report.

```
sbt coverage test
sbt coverageReport
```

Note that `coverageReport` can't be put in the same command as `coverage`.


It has left me thinking whether my fight with SBT is worthwhile.
 





