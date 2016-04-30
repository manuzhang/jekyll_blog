---
title: quick start in cvs
comments: true
layout: post
---

**CVS**, concurrent version system, which I firstly came across in my pintos project. Actually, I thought it would be a burden while I regret not having employ the strength of a simple version system. My laziness left me with only one version of pintos and I can't recall what changes I've made along the way. If you're trying to do something seriously, a version control system is really needed, centralized like **CVS**, **SVN**, distributed like **GIT**. To set out, **CVS** may be a good choice for its simplicity.

All you need are two directories, **Repository** and **Sandbox**. Repository is your backup. Sandbox is where you are free to mess stuff up. When you've made changes in Sandbox you commit to Repository. Also, any changes in Repository(codes submitted by others, for instance) can update to Sandbox.

Now, let's kick off!

Firstly, create the **Repository**

```bash
$ su -
Password: ******

$ mkdir /var/lib/cvsroot
$ chgrp anthill /var/lib/cvsroot
$ ls -la /var/lib

total 153
drwxr-xr-x 2 root anthill 4096 Jun 28 16:31 cvsroot

$ chmod g+srwx /var/lib/cvsroot
$ ls -la /var/lib

total 153
drwxrwsr-x 2 root anthill 4096 Jun 28 16:33 cvsroot

$ cvs -d /var/lib/cvsroot init
$ ls -la /var/lib

total 3
drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 cvsroot

$ ls -la /var/lib/cvsroot

total 12
drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 .
drwxr-xr-x 10 root staff 4096 Jun 28 16:35 ..
drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 CVSROOT

$ chown -R cvs /var/lib/cvsroot
```

Secondly, import a project

```bash
$ mkdir example
$ touch example/file1
$ touch example/file2
$ cd example
$ cvs -d /var/lib/cvsroot import example example*project ver*0-1
```

now, checkout

```bash
$ mkdir ~/cvs
$ cd ~/cvs
$ cvs -d /var/lib/cvsroot checkout example
$ cd ~/cvs/example
$ cvs commit
cvs commit: Examining
```

update the **Sandbox**

```bash
$ cvs update -d

cvs update: Updating .
U file2
cvs update: Updating directory

$ ls

CVS directory file1 file2
```

We're almost done here. You may also set your default cvs editor by keeping a **.cvsrc** in the **home** directory.
I use emacs so put the following line in my **.cvsrc**

`setenv CVSEDITOR emacs`

All the changes will be shown to you via the editor. Of course, you're able to write logs with your favorite editor.
All the commands are copied from the book *Essential CVS, 2nd Edition*.

