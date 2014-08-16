---
author: manuzhang
comments: true
date: {}
layout: post
slug: "maven-at-work"
title: Maven at work
wordpress_id: 1115
categories: 
  - maven
  - project management
published: true
---

<blockquote>
Apache Maven is a software project management and comprehension tool. Based on the concept of a project object model (POM), Maven can manage a project's build, reporting and documentation from a central piece of information.
</blockquote>

The concept of "software project management" (SPM) has never occurred to me till I started my career and worked on Hadoop at Intel. The Hadoop community was shifting from [Ant](http://ant.apache.org/) to [Maven](http://maven.apache.org/index.html) then so Maven happened to be the first SPM tool I tried to make sense of. When starting out, my learning materials mainly came from 3 sources, manuals on Maven website, [Maven: The definitive Guide](http://www.ppurl.com/2009/12/maven-the-definitive-guide.html) and other tutorials on the web.

_The Definitive Guide_ is a great for kicking off, to learn the basics (how Maven works, the concepts of lifecycle and goal, etc) and start my first Maven project but not far from here. The examples in the book don't apply to projects in my work. And they are not supposed to be. When running into a problem at work, it is much faster to Google it, which usually leads to the official manuals. The book is better for doing homework afterwards.

As for manuals, they are good reference when I already know which weapon to use. What if I don't ? For example, I want to manage test with maven but don't know about the surefire plugin. I usually go through the pom file of a project which works similar to mine, look for the specific plugin or configuration that does the magic and copy it. After several turns of trials and errors, I finally find my answers. Another problem of manual is that it is too verbose. I have to skip several paragraphs before finding a solution that may work. I wish I had a concise tutorial telling me what weapons to use according to the situations.

Meanwhile, I'd like to write down how I solved the problems I've come across at work with maven. Hence, I decided to start a series of posts called _Maven at work_ which would be organized around common usages (compile, test, distribution) in my daily work.