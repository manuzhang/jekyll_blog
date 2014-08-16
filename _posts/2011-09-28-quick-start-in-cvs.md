---
author: manuzhang
comments: true
date: 2011-09-28 11:55:05+00:00
layout: post
published: false
slug: quick-start-in-cvs
title: quick start in cvs
wordpress_id: 49
categories:
- linux
- version_control
tags:
- cvs
- version_control
---

_CVS_, current version system, which I firstly came across in my pintos project.

Actually, I thought it would be a burden while I regret not having employ the strength of a simple version system.

My laziness left me with only one version of pintos and I can't recall what changes I've made along the way.

If you're trying to do something seriously, a version control system is really needed, centralized like _CVS_, _SVN_, distributed like _GIT_.



To set out, _cvs_ may be a good choice for its simplicity.



<!-- more -->

All you need are two directories, _Repository_ and _Sandbox_. _Repository_ is your backup. _Sandbox_ is where you are free to mess stuff up. When you've made changes in _Sandbox_ you _commit_ to _Repository_. Also, any changes in _Repository_(codes submitted by others, for instance) can _update_ to _Sandbox_.



Now, let's kick off!



Firstly, create the _Repository_

$ su -

Password: ***|\_*_|





# mkdir /var/lib/cvsroot





# chgrp anthill /var/lib/cvsroot





# ls -la /var/lib



total 153

drwxr-xr-x 2 root anthill 4096 Jun 28 16:31 cvsroot





# chmod g+srwx /var/lib/cvsroot





# ls -la /var/lib



total 153

drwxrwsr-x 2 root anthill 4096 Jun 28 16:33 cvsroot





# cvs -d /var/lib/cvsroot init





# ls -la /var/lib



total 3

drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 cvsroot





# ls -la /var/lib/cvsroot



total 12

drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 .

drwxr-xr-x 10 root staff 4096 Jun 28 16:35 ..

drwxrwsr-x 3 root anthill 4096 Jun 28 16:56 CVSROOT





# chown -R cvs /var/lib/cvsroot



Secondly, import a project



/tmp$ mkdir example

/tmp$ touch example/file1

/tmp$ touch example/file2

/tmp$ cd example

/tmp/example$ cvs -d /var/lib/cvsroot import example example_project ver_0-1



now, checkout



$ mkdir ~/cvs

$ cd ~/cvs

$ cvs -d /var/lib/cvsroot checkout example



commit changes



$ cd ~/cvs/example

$ cvs commit

cvs commit: Examining



update the _Sandbox_



$ cvs update -d

cvs update: Updating .

U file2

cvs update: Updating directory

$ ls

CVS directory file1 file2



We're almost done here



you may also set your default cvs editor by keeping a _.cvsrc_ in the _home_ directory.



I use emacs so put the following line in my _.cvsrc_.



`setenv CVSEDITOR emacs`

All the changes will be shown to you via the editor. Of course, you're able to write logs with your favorite editor.



All the commands are copied from the book _Essential CVS, 2nd Edition_.



