---
id: fReYVyqwWjpWNGn8f3hSy
title: Autogenning Node.js
desc: ''
updated: 1643143681761
created: 1643132654642
excerpt: >-
  Writing an autogen script for Node.js from scratch.
---

![nodejs-logo-part-2](/assets/images/nodejs-logo-part-2.png)

In the [[previous|blog.oss.funtoo.building-nodejs-from-scratch]] blog post about
Node.js, we wrote an ebuild for it from scratch. In this post, we'll look into
how to take the existing fixed version ebuild and convert it to an autogen!

Our goal for this post is to write an autogen script, which automatically
figures out what the latest release of Node.js is and generates an ebuild for
it. For simplicity, we'll target the latest current release of Node.js since it
will make things a bit easier.

As mentioned previously, autogens are powered by the
[metatools](https://code.funtoo.org/bitbucket/users/drobbins/repos/funtoo-metatools/browse)
framework. Our first step should be to install metatools in our development
environment. In my case, I'll be using the existing LXD container I made for the
last post.

We're interested in installing the [latest development
sources](https://code.funtoo.org/bitbucket/users/drobbins/repos/funtoo-metatools/browse/docs/install.rst#47).
At the time of writing this, the metatools ebuild in the Funtoo kits doesn't
work, but we can still use it to instruct Portage to pull in the necessary
dependencies.

So, let's run `emerge --onlydeps metatools` and give it a moment to finish.
After that, we also need to install [MongoDB](https://www.mongodb.com), since
it's used by metatools for persistence of various things: `emerge mongodb`.

A couple of massive compilations later, we can proceed with the metatools
installation. All that's left is to clone the repos and set a few environment
variables:

```sh
$ git clone ssh://git@code.funtoo.org:7999/~drobbins/funtoo-metatools.git
$ git clone ssh://git@code.funtoo.org:7999/~drobbins/subpop.git
$ cat >> .bashrc <<EOF
export PATH=$HOME/funtoo-metatools/bin:$PATH
export PYTHONPATH=$HOME/subpop:$HOME/funtoo-metatools
EOF
$ source .bashrc
$ doit --help
```

This should be the entire metatools setup covered. All that's left is to start
MongoDB and we're ready to go: `/etc/init.d/mongodb start`.

Let's go back in `nodejs-overlay/net-libs/nodejs` and turn our ebuild into a
template:

```sh
$ mkdir templates
$ mv nodejs-16.13.2.ebuild templates/nodejs.tmpl
```

We can also wipe the manually generated Manifest file since metatools will
automatically generate it for us when we run the autogen.

We'll get back to the template once we flesh out our autogen script, which is
precisely what we're going to get into now. We need to make an `autogen.py` file
and define a `generate` async function in it like so:

```python
#!/usr/bin/env python


async def generate(hub, **pkginfo):
    pass
```

I won't get into the details of how metatools works - for that you can refer to
the documentation. What we need to know is that `generate` acts as the entry
point of the autogen script. I'll also briefly explain what the two arguments we
receive in `generate` are:
- `hub`: core paradigm of Plugin-Oriented Programming; it is used to call
  metatools code.
- `pkginfo` keyworded argument list, which contains info about the package we're
  generating (e.g. name, category, template directory, etc).

BTW, running an autogen script is as simple as running `doit` in its directory.
We'll be running `doit` quite often to see if things are going as expected.
  
The first thing our autogen should do is to figure out what the latest version
for Node is. Since we're already using GitHub to pull in the sources, the best
way to figure this out is via the GitHub API.

For this approach to work, the upstream repo should either have tags or
releases. In the case of Node, the upstream uses
[releases](https://github.com/nodejs/node/releases).

By looking at the [GitHub API
reference](https://docs.github.com/en/rest/reference/releases#list-releases), we
can figure out that the endpoint we're after is
`/repos/{owner}/{repo}/releases`. Let's try to fetch it in the autogen.

First things first, we need to know the owner and repository names. Let's store
them in two variables:

```python
async def generate(hub, **pkginfo):
    github_user = "nodejs"
    github_repo = "node"
```

Then, we need to build the URL and send a request. The metatools way of doing
this is via the `hub.pkgtools.fetch.get_page()` function:

```python
    releases = await hub.pkgtools.fetch.get_page(
        f"https://api.github.com/repos/{github_user}/{github_repo}/releases",
        is_json=True,
    )
```

As you can tell, the only positional argument to `get_page` is a URL. There are
also various optional arguments, but the most useful one is `is_json` which
parses the response as JSON for you.

The endpoint is supposed to return a list of objects representing the releases.
We can use this list to figure out what the latest release is.

While, intuitively, it might make sense for the first release in the response to
be the latest one, it's not always the case, and it's better to be safe than
sorry.

There might be draft releases or prereleases at the start of the list, and we
definitely don't want to package those. In the case of Node, there might also be
a LTS release created after the latest current release.

What we usually do is we make use of
[`packaging.version`](https://packaging.pypa.io/en/latest/version.html) to
figure out what the newest version is. Let's import it in our autogen:

```python
from packaging import version
```

In addition to that, we also filter out all draft and prerelease releases.

There is also a theoretical case in which we can't find a suitable release to
package at all. While it is practically unlikely, it's still a good idea to
handle it just in case.

The magic words to do this in Python look like this:

```python
    try:
        latest_release = max(
            (
                release
                for release in releases
                if not release["prerelease"] and not release["draft"]
            ),
            key=lambda release: version.parse(release["tag_name"]),
        )
    except ValueError:
        raise hub.pkgtools.ebuild.BreezyError(
            f"Can't find suitable release of {github_repo}"
        )
```

This fragment does exactly what we described above: it finds the latest release
in terms of version, which isn't a draft or a prerelease. In the case in which
`max` finds nothing, an error is thrown, so we catch that and rethrow something
with a more meaningful error message.

To verify it works, we can try logging `latest_release[tag_name]`. As expected,
the result is `v17.4.0` (which is the latest at the time of writing this).

Now that we have the latest release, we can use it to retrieve the source
tarball URL and the actual version we're packaging. As you may have noticed
already, the version of a release is its `tag_name`. As for the tarball, it's
URL is the `tarball_url`.

Also, notice that the tag name has a prefix of `v`. Portage wouldn't really like
that prefix, so we should strip it before generating an ebuild for it:

```python
    latest_version = release["tag_name"].lstrip("v")
    latest_tarball = release["tarball_url"]
```

Now that we have the tarball, we need to also create an "artifact" for it. An
artifact in metatools is a resource that is used by a `BreezyBuild`, and
ultimately referenced in an ebuild.

The artifacts are precisely the info metatools uses to figure out what to put in
the Manifest. They are also what we use to set the `SRC_URI` in the template.

Let's create an artifact by specifying the URL and final name for it:

```python
    tarball_artifact = hub.pkgtools.ebuild.Artifact(
        url=latest_tarball, final_name=f"{pkginfo['name']}-{latest_version}.tar.gz"
    )
```

All that's left is to create the `BreezyBuild` and `push` it. `BreezyBuild`
basically wraps the context for the ebuild generation. We need to pass the
`pkginfo`, the version, the artifacts, along with any other values we'd want to
access in the template:

```python
    ebuild = hub.pkgtools.ebuild.BreezyBuild(
        **pkginfo,
        version=latest_version,
        artifacts=artifacts,
    )

    ebuild.push()
```

This is basically the whole autogen script! All that's left is to tweak the
template and test everything out. Let's begin by replacing the hardcoded
`SRC_URI` with the URI from the tarball artifact:

```jinja
SRC_URI="{{ artifacts[0].src_uri }}"
```

It's as simple as accessing the first (and only) element in the `artifacts`
list's `src_uri` property. And since we're not doing anything special in the
rest of the template, this should theoretically be good enough. Let's give it a
try!

Let's run the autogen and look at the generated ebuild. This is how it looks for
me:

```bash
# Distributed under the terms of the GNU General Public License v2

EAPI=7

PYTHON_COMPAT=( python3+ )

inherit python-any-r1

DESCRIPTION="Node.js JavaScript runtime"
HOMEPAGE="https://nodejs.org"
SRC_URI="https://api.github.com/repos/nodejs/node/tarball/v17.4.0 -> nodejs-17.4.0.tar.gz"

LICENSE="Apache-1.1 Apache-2.0 BSD BSD-2 MIT"
SLOT="0"
KEYWORDS="*"
IUSE=""

DEPEND=""
RDEPEND="${DEPEND}"
BDEPEND="
	${PYTHON_DEPS}
"

post_src_unpack() {
	mv "${WORKDIR}"/node-"${PV}" "${S}" || die
}

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

It's exactly like the ebuild we wrote manually, with the only exception being
the `SRC_URI`. We can also see that a Manifest was generated for us in the
process. Neat!

To see if it works, let's `emerge` it! And... it failed. Oh well, it's the source
filenames again:

```
mv: cannot stat '/var/tmp/portage/net-libs/nodejs-17.4.0/work/node-17.4.0': No such file or directory
```

Turns out that the tarball we got from the release has a different structure
than the one we used before. Let's investigate.

Peeking into `/var/tmp/portage/net-libs/nodejs-17.4.0/work`, we find a directory
called `nodejs-node-eeed0bd`. The name of the directory is actually in the
format `{github_user}-{github_repo}-{commit_sha}`.

Conveniently, we already have the GitHub user and repo defined in the autogen
script. And as I mentioned earlier, we can pass additional arguments to the
`BreezyBuild` if we need them for the template:

```python
    ebuild = hub.pkgtools.ebuild.BreezyBuild(
        **pkginfo,
        version=latest_version,
        github_user=github_user,
        github_repo=github_repo,
        artifacts=[tarball_artifact],
    )
```

Note that we don't have the commit SHA, but we don't really need it, since it's
fine to just use a glob for it. Now, we can use these two in the template:

```bash
post_src_unpack() {
	mv "${WORKDIR}"/{{ github_user }}-{{ github_repo }}-* "${S}" || die
}
```

Let's generate the ebuild again. Peeking at the ebuild, we see exactly what we
expected in `post_src_unpack`:

```
post_src_unpack() {
	mv "${WORKDIR}"/nodejs-node-* "${S}" || die
}
```

This looks promising so far, so let's `emerge` away! And it should be no surprise that it worked!

```sh
$ node
Welcome to Node.js v17.4.0.
Type ".help" for more information.
> console.log('Hello, world!')
Hello, world!
undefined
```

We now have a script that will always package the latest version of Node for us!
If this sounds cool, in the future, we'll look into packaging multiple Node
versions at once, precisely the LTS ones, which is even cooler in my opinion.
Stay tuned!

As always, the final code can be found somewhere in the commit history of the
[nodejs-overlay](https://code.funtoo.org/bitbucket/users/invakid404/repos/nodejs-overlay/browse)
repo.