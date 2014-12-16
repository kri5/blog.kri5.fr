---
layout: post
status: publish
published: true
title: Ceph starting guide
comments: true
tags:
- Ceph
- Dev
- FOSS
- Howto
---
Hi everyone,

I recently started working on [Ceph](http://ceph.com).

First of all, Ceph is a collection of programs that provides Object storage, Block Storage and Distributed FileSystem, with no [SPOF](http://en.wikipedia.org/wiki/Single_point_of_failure). For a more detailled overview, checkout this [Slides](http://www.slideshare.net/Inktank_Ceph/inktankceph-overview).

The code is available on [GitHub](http://github.com/ceph/ceph).

If, as i do, you are new to ceph, you should start by compiling ceph from sources.

    ./autogen.sh && ./configure && make

Once compiled, you're up and running to start contributing!

But wait... How do you test your code? Do i have to deploy a whole cluster? Yes! and no!

A tiny hidden undocumented tool is here to help you in this task : `vstart.sh` \o/

This tool allows you to deploy a whole fake cluster on your filesystem, with various options, and also some environment variables can configure the number of OSD/MDS/MON/RGW you want to run.

You can also find its sibling called `stop.sh` to stop all processes launched by `vstart.sh`

Hope this little post will help you in getting started with Ceph contribution.

PS: A man page for vstart is on its way out. ;)
