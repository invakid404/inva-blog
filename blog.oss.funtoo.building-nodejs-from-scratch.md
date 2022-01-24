---
id: emNwzI1Dq0weEz0eXLEsK
title: Building Node.js from Scratch
desc: ''
updated: 1643056824189
created: 1643044468027
excerpt: >-
  Funtooizing the Node.js ebuild.
---

![nodejs-logo](/assets/images/nodejs-logo.png)

As a Funtoo user who uses [Node.js](https://nodejs.org) at work, I thought it'd
be great to have the latest and greatest version of Node readily available to
me. That's why I decided it'd be a great idea to "autogen" it.

For those who are unfamiliar with the concept of autogens in Funtoo, they are
practically our very own robot package maintainers. With the power of the
[metatools](https://code.funtoo.org/bitbucket/users/drobbins/repos/funtoo-metatools/browse)
framework, we can write scripts, which automagically generate ebuilds for us.

Rewriting the ebuild from scratch isn't a prerequisite for writing an
autogen, but since the Node.js ebuild is unfamiliar territory to me,
I thought I might as well try and strip all the nonsense from it. It was also an
opportunity to document how I personally approach packaging software from
scratch.

This blog post captures the process of figuring out how Node.js is built and
putting together a brand new (and hopefully easier to understand) ebuild for it.

I set up a quick Funtoo LXD container for testing since I don't feel like
breaking the Node installation on my host. Now, it's time to set up a local
overlay as per the [docs](https://www.funtoo.org/Creating_Your_Own_Overlay). I
created a fork of the Skeleton overlay repo and called it "nodejs-overlay".

After setting up the development environment, it's time for the main dish. At
the time of writing this, the latest LTS version of Node is 16.13.2, so it will
be the one we're targeting. Also, all the URL references to the Node.js GitHub
repo are for a specific commit, so this post should still make sense even if
things change in the future.

Let's create a new file in the `nodejs-overlay/net-libs/nodejs` directory,
called `nodejs-16.13.2.ebuild` and fill in the basic stuff:

```bash
# Distributed under the terms of the GNU General Public License v2

EAPI=7

DESCRIPTION="Node.js JavaScript runtime"
HOMEPAGE="https://nodejs.org"
SRC_URI="https://github.com/nodejs/node/archive/refs/tags/v16.13.2.tar.gz -> ${P}.tar.gz"

LICENSE="Apache-1.1 Apache-2.0 BSD BSD-2 MIT"
SLOT="0"
KEYWORDS="*"
IUSE=""

DEPEND=""
RDEPEND="${DEPEND}"
BDEPEND=""
```

Now, we can run `ebuild nodejs-16.13.2.ebuild digest` to pull in the sources and
generate a `Manifest` file.

It's about time to look into how Node.js builds.
[This](https://github.com/nodejs/node/blob/ce41395f89414dfd459084ea61a7eeac1f67713a/BUILDING.md)
document in the repo appears to be a good starting point. Scrolling down to the
"building on Unix" section, we can see that Node uses GNU make to build, and
additionally requires Python 3 to be installed. 

We don't really need to worry about GNU make being present, so we can go ahead
and add a dependency on Python 3+ for now. We will do it by inheriting the
[python-any-r1](https://devmanual.gentoo.org/eclass-reference/python-any-r1.eclass/index.html)
eclass:

```bash
inherit python-any-r1
```

Before the inherit line, we should also define `PYTHON_COMPAT`:

```bash
PYTHON_COMPAT=( python3+ )
```

*Note*: both of these should go under the `EAPI` line, since eclasses usually
require that to be set in advance.

Then, we can add `${PYTHON_DEPS}` to `BDEPEND`:

```bash
BDEPEND="
    ${PYTHON_DEPS}
"
```

The best way to tell if we've done anything meaningful is to simply try to
emerge our newly created ebuild. While we definitely don't expect it to work,
the error we get will give us a clue what to do next.

Doing exactly that, I immediately get hit with the following error:

```
 * ERROR: net-libs/nodejs-16.13.2::nodejs-overlay failed (prepare phase):
 *   The source directory '/var/tmp/portage/net-libs/nodejs-16.13.2/work/nodejs-16.13.2' doesn't exist
```
 
Seems like our source directory is called something else. Let's investigate!

We can run `ebuild nodejs-16.13.2.ebuild clean unpack` to wipe the temporary
files for nodejs and unpack the source code. Then, we can look into
`/var/tmp/portage/net-libs/nodejs-16.13.2/` to see what things look like.

The source directory lives in the `work/` subdirectory. Running `ls -la` in
there produces the following output:

```
total 0
drwx------ 1 portage portage  24 Jan 24 19:19 .
drwx------ 1 portage portage 146 Jan 24 19:19 ..
drwxr-xr-x 1 portage portage 820 Jan 10 20:02 node-16.13.2
```

Yep, indeed we need to tweak the source directory name. We can either override
the `S` (path to temporary build directory/source directory) variable in the
ebuild, or we can move the directory to `S`. In Funtoo, we tend to prefer the
latter.

To achieve this, we can add `post_src_unpack` to our ebuild and perform a move
in it like so:

```bash
post_src_unpack() {
    mv "${WORKDIR}"/node-"${PV}" "${S}" || die
}
```

Simple as that! If we try to emerge nodejs again, we get a tiny bit further:

```
 * nodejs-16.13.2.tar.gz BLAKE2B SHA512 size ;-) ...                                                                                                                                     [ ok ]
 * Using python3.7 to build
>>> Unpacking source...
>>> Unpacking nodejs-16.13.2.tar.gz to /var/tmp/portage/net-libs/nodejs-16.13.2/work
>>> Source unpacked in /var/tmp/portage/net-libs/nodejs-16.13.2/work
>>> Preparing source in /var/tmp/portage/net-libs/nodejs-16.13.2/work/nodejs-16.13.2 ...
>>> Source prepared.
>>> Configuring source in /var/tmp/portage/net-libs/nodejs-16.13.2/work/nodejs-16.13.2 ...
 * econf: updating nodejs-16.13.2/deps/cares/config.guess with /usr/share/gnuconfig/config.guess
 * econf: updating nodejs-16.13.2/deps/cares/config.sub with /usr/share/gnuconfig/config.sub
./configure --prefix=/usr --build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --mandir=/usr/share/man --infodir=/usr/share/info --datadir=/usr/share --sysconfdir=/etc --localstatedir=/var/lib --libdir=/usr/lib64
Node.js configure: Found Python 3.7.10...
gyp: --mandir=/usr/share/man not found (cwd: /var/tmp/portage/net-libs/nodejs-16.13.2/work/nodejs-16.13.2) while trying to load --mandir=/usr/share/man
Error running GYP
```

Seems like we hit the configure phase. This is where we need to tweak the
options to get things to work. The issue in this case appears to be that `gyp`
is failing to find the `mandir`.

This is one of the default flags that is set by `econf`, and `gyp` for some
reason doesn't like it, so I suppose we're better off overriding `src_configure`
in our ebuild (we'll have to do it later anyway) and calling the configure
script manually:

```bash
src_configure() {
    ./configure
}
```

Indeed, this gets us past the configuration phase and we're now in business!
Node is compiling!

While we're waiting, we might as well go back and add a little comment to note
why we chose not to use the `econf` wrapper:

```bash
src_configure() {
    # NOTE: `econf` default flags appear to trip up the configure process,
    #       directly call the `./configure` script instead.
    ./configure
}
```

And after a hefty compile, it indeed was this simple! We now have a functional
Node.js installation, compiled from source. It is installed outside of `PATH`
though (at least on a standard Funtoo system): `/usr/local`. 

We can still verify that the freshly compiled binary works. Let's run it by
specifying the full path:

```sh
$ /usr/local/bin/node 
Welcome to Node.js v16.13.2.
Type ".help" for more information.
> console.log('Hello, world!')
Hello, world!
undefined
```

And indeed, it does! This is certainly good enough for this blog post. As a
final touch, let's make it install in `/usr` instead of `/usr/local`. To do
this, you usually need to specify some sort of `prefix` while configuring. The
way you pass this flag differs from build system to build system, so we'll need
to check where Node expects it.

The logical place to start our investigation is the
[`configure`](https://github.com/nodejs/node/blob/ce41395f89414dfd459084ea61a7eeac1f67713a/configure)
script. It appears to just be checking our Python version and importing the
`configure` Python module if it's supported. Otherwise, it just throws some
fancy error we don't care about.

So, the real configuration appears to be happening in
[`configure.py`](https://github.com/nodejs/node/blob/ce41395f89414dfd459084ea61a7eeac1f67713a/configure.py).
Skimming through it, we can spot the `--prefix` option
[here](https://github.com/nodejs/node/blob/ce41395f89414dfd459084ea61a7eeac1f67713a/configure.py#L77).
It appears to be exactly what we're after!

Let's just override it in `src_configure` then. While we can simply set it to
`/usr` and call it a day, it doesn't hurt to also prepend `EPREFIX` to it. Since
Portage can technically be running in a "prefix" (not targeting /), it's "good
practice" to account for that in the ebuild.

This is how our `src_configure` looks now:

```bash
src_configure() {
    configure_options=(
        # By default, prefix is /usr/local, which is outside of PATH,
        # set it to /usr instead:
        --prefix="${EPREFIX}"/usr
    )

    # NOTE: `econf` default flags appear to trip up the configure process,
    #       directly call the ./configure script instead.
    ./configure "${configure_options[@]}"
}
```

With that, we have a pretty satisfactory result. We'll look into autogenning
Node.js and potentially extending this ebuild in the future.

BTW, the ebuild we wrote in this blog post can be found in
[this](https://code.funtoo.org/bitbucket/users/invakid404/repos/nodejs-overlay/browse)
repo.