---
id: kl74li714iahob4f78rphx0
title: Having Fun with CLFS and musl
desc: ''
updated: 1645651927059
created: 1645649115079
excerpt: >-
  Experimenting with building a CLFS prefix with musl instead of glibc.
---

![bootstrapping](/assets/images/bootstrapping.png)

I've been playing around with CLFS over the last week or two, and it has been
super fun. Well, that's at least when things worked :D

More specifically, I've been experimenting with replacing glibc with musl and
seeing how complicated it would be to get running. And, spoiler alert,
considering you're currently reading this, it's safe to assume I've had a fair
bit of success with it.

You might be wondering how Funtoo comes into the story, and I got the answers.
We recently announced a new Funtoo project called `Evolved Bootstrap`. The idea
behind it is to develop a way to bootstrap a Funtoo Linux system completely from
scratch, using any Linux-like environment, for any architecture. The end goal is
to have a straightforward and automated way to port Funtoo for any currently
unsupported architecture.

One cannot fly into flying, though, so we need to learn how to walk first: we
need a way of bootstrapping a minimal cross-compiled environment that we can
expand upon later. And this is basically what the concept of CLFS is: building a
Linux system from scratch for another architecture. Right now, we're running
through the CLFS books and seeing what it takes to get things running for
different architectures, with various package upgrades, etc.

The coolest part about the Evolved Bootstrap, in my opinion, is that it will
also unlock the doors to alternative init systems, libc implementations,
compilers, etc. That's why I decided to experiment with replacing musl with
glibc. The CLFS embedded book already used musl, but as the name may suggest, it
caters to building Linux systems for embedded devices and doesn't cover our use
case.

The [musl project](https://musl.libc.org) has always been super interesting to
me, but I've never had a real chance to play around with it on Funtoo, and it
has never been a top priority for us to support. The Evolved Bootstrap project
gave me a valid excuse to dive right into it, though.

By mixing and matching the standard and embedded CLFS books and through
countless workarounds, I've managed to get a chrootable environment going with
musl instead of glibc. You can find my instructions on the Funtoo wiki
[here](https://www.funtoo.org/User:Invakid404/CLFS). They are a more concise and
copy-and-paste-ready version of the CLFS book, and I've tested them for `x86_64`
and `aarch64` so far. I'm currently working on getting them to work for
`powerpc64` as well.

If the Evolved Bootstrap project sounds like fun to you, you can learn more
about it [here](https://www.funtoo.org/Funtoo:Evolved_Bootstrap), and you can
stop by our Discord if you feel like chatting to us about it and perhaps even
getting involved.