---
id: mtOTe8lbUvuixBVzLr9ZE
title: Enhancing the Node.js Autogen
desc: ''
updated: 1643230811737
created: 1643224539828
excerpt: >-
  Generating ebuilds for all Node.js release channels at once.
---

![nodejs-logo-part-3](/assets/images/nodejs-logo-part-3.png)

In the [[previous|blog.oss.funtoo.autogenning-nodejs]] blog post about Node.js,
we wrote an autogen script for it from scratch. Today, we'll look into extending
our code to generate ebuilds for all Node release channels at once!

This is needed since not everything works with every Node version. And
sometimes, you're just stuck with a particular version. My goal is to avoid
having to resort to "Node version managers" or anything like that and just have
the ability to install any Node version directly via Portage.

To achieve this, we'll need to combine a couple of sources of information.
Firstly, we'll need to fetch all releases and group them by major version via
the GitHub API, similarly to how we're doing it now. We'll also need to fetch a
list of LTS release channels to know which Node versions to unmask by default.

The latter is needed so that we only offer LTS releases to the users by default,
but still leave a way to install latest current, or even a discontinued version
if that's what you're after, I'm not judging :D
    
In particular, we can fetch
[this](https://github.com/nodejs/Release/blob/main/schedule.json) JSON file from
the `Releases` repository in the Node organization. It gives us a timeline of
all Node releases, and we can use that to figure out whether a release is still
supported and, more importantly, whether it's an LTS release.

Let's begin with the fetching of GitHub releases. If we pay close attention to
the [API
reference](https://docs.github.com/en/rest/reference/releases#list-releases), we
can see that the "list releases" is in fact paginated. Since we want to fetch
all releases, we'll have to fetch all pages.

The most convenient way to do this in my opinion is to write an [`async
generator`](https://www.python.org/dev/peps/pep-0525/). Its purpose will be to
query the GitHub API until there are pages left and yield us the releases it
receives. We'll later use that information to find the latest release per
release channel.

My implementation of this generator looks like this:

```python
async def release_generator(hub, github_user, github_repo):
    page = 0
    while True:
        releases = await hub.pkgtools.fetch.get_page(
            f"https://api.github.com/repos/{github_user}/{github_repo}/releases?page={page}&per_page=100",
            is_json=True,
        )

        if not releases:
            break

        for release in releases:
            yield release

        page += 1
```

In an infinite loop, it queries the GitHub API for the releases at the current
page. If we receive nothing, this means we've reached the end of the releases,
so we can terminate. Otherwise, it yields all received releases and moves on to
the next page. Nothing too fancy.

Note that we're also specifying the `per_page` query parameter. It's set to
`100` (the maximum) to minimize the amount of requests we make to GitHub.

With that out of the way, we can now iterate all releases like this:

```python
    async for release in release_generator(hub, github_user, github_repo):
        pass
```

Simple as that! Now, let's define a dictionary, in which we'll store the latest
release for every release channel up to this point. The high-level description
of the algorithm for figuring out what the latest release for every major
version looks like this:
- Parse the current release's version to get its major version.
- Check the dictionary to see what the latest release for this major version is
  up to this point.
- If nothing is found or if the current version is newer than the version of the
  stored release, update the dictionary.
- Otherwise, proceed with the next release.

Translated to Python, the algorithm looks like this:

```python
    latest_release_by_major = {}

    async for release in release_generator(hub, github_user, github_repo):
        release_version = release["version"] = version.parse(release["tag_name"])

        release_major = str(release_version.major)
        latest_release = latest_release_by_major.get(release_major)

        if latest_release is None or latest_release["version"] < release_version:
            latest_release_by_major[release_major] = release
```

If we inspect the versions of all the releases in the dictionary after the loop,
we get something that looks like this:

```
Release channel: 17; Latest version: 17.4.0
Release channel: 16; Latest version: 16.13.2
Release channel: 14; Latest version: 14.18.3
Release channel: 12; Latest version: 12.22.9
Release channel: 15; Latest version: 15.14.0
Release channel: 10; Latest version: 10.24.1
Release channel: 13; Latest version: 13.14.0
Release channel: 8;  Latest version: 8.17.0
Release channel: 11; Latest version: 11.15.0
Release channel: 6;  Latest version: 6.17.1
Release channel: 9;  Latest version: 9.11.2
Release channel: 4;  Latest version: 4.9.1
```

Perfect! All that's left to do is to figure out which of those versions we want
to unmask. So let's fetch the release schedule JSON and parse the `end` dates
within it.

To parse the dates, we'll need to import `datetime`:

```python
from datetime import date
```

Now, for the actual fetching and parsing of the release schedule:

```python
    release_schedule = await hub.pkgtools.fetch.get_page(
        f"https://raw.githubusercontent.com/{github_user}/Release/main/schedule.json",
        is_json=True,
    )

    today = date.today()
    for release_channel, schedule in release_schedule.items():
        major_version = release_channel.lstrip("v")
        release = latest_release_by_major.get(major_version)

        if release is None:
            continue

        if "lts" not in schedule:
            continue

        end_date = date.fromisoformat(schedule["end"])
        if today > end_date:
            continue

        release["unmasked"] = True
```

As mentioned above, we fetch the release schedule, we iterate through every
release channel in it, we try to find the corresponding release in our
dictionary, then we check if it's an LTS release at all, and lastly we check
whether it has reached its EOL.

If all three of those conditions hold, we mark the release as "unmasked" by
setting the corresponding key to `True`.

Armed with that information, we can simply loop through the dictionary and
generate an ebuild for every single one. To make this a bit nicer, let's extract
the actual ebuild generation to a separate function:

```python
def generate_for_release(hub, release, **pkginfo):
    release_version = release["tag_name"].lstrip("v")
    tarball_url = release["tarball_url"]

    tarball_artifact = hub.pkgtools.ebuild.Artifact(
        url=tarball_url, final_name=f"{pkginfo['name']}-{release_version}.tar.gz"
    )

    ebuild = hub.pkgtools.ebuild.BreezyBuild(
        **pkginfo,
        version=release_version,
        artifacts=[tarball_artifact],
        unmasked="unmasked" in release,
    )

    ebuild.push()
```

Nothing special here, except for the new "unmasked" parameter in `BreezyBuild`.
As you may have noticed, I've removed `github_user` and `github_repo` from the
`BreezyBuild`. We can just set those in `pkginfo` to avoid having to pass them
as separate arguments every time:

```python
    github_user = pkginfo["github_user"] = "nodejs"
    github_repo = pkginfo["github_repo"] = "node"
```

With that, our autogen already works! The only thing left is to tweak the
template to take `unmasked` into account:

```bash
KEYWORDS="{{ '*' if unmasked else '' }}"
```

And we're pretty much done! Let's run the autogen and check whether the keywords
are as we expect:

```sh
$ grep -r "KEYWORDS="
templates/nodejs.tmpl:KEYWORDS="{{ '*' if unmasked else '' }}"
nodejs-17.4.0.ebuild:KEYWORDS=""
nodejs-8.17.0.ebuild:KEYWORDS=""
nodejs-11.15.0.ebuild:KEYWORDS=""
nodejs-10.24.1.ebuild:KEYWORDS=""
nodejs-12.22.9.ebuild:KEYWORDS="*"
nodejs-16.13.2.ebuild:KEYWORDS="*"
nodejs-6.17.1.ebuild:KEYWORDS=""
nodejs-4.9.1.ebuild:KEYWORDS=""
nodejs-13.14.0.ebuild:KEYWORDS=""
nodejs-15.14.0.ebuild:KEYWORDS=""
nodejs-14.18.3.ebuild:KEYWORDS="*"
nodejs-9.11.2.ebuild:KEYWORDS=""
```

And indeed, they are! We can try to unmask and emerge an older version of Node
and see whether our template works for it as well:

```sh
# mkdir -p /etc/portage/package.accept_keywords
# cat > /etc/portage/package.accept_keywords/nodejs <<EOF
net-libs/nodejs **
EOF
# emerge -a =nodejs-6.17.1
$ node --version
v6.17.1
```

Piece of cake!

*NOTE*: There is actually an issue with Node v4 in particular: it uses Python 2
to build. While we could just drop it, it's equally as simple to make it build
with Python 2 as well. Since we still have Python 2 at the time of writing this,
why not?

According to the
[changelogs](https://github.com/nodejs/node/blob/master/doc/changelogs/CHANGELOG_V6.md),
the first version to support Python 3 was Node v6, so let's add a
`python_compat` variable to the `BreezyBuild` to fix this:

```python
    ebuild = hub.pkgtools.ebuild.BreezyBuild(
        **pkginfo,
        version=release_version,
        artifacts=[tarball_artifact],
        unmasked="unmasked" in release,
        # NOTE: First version to support Python 3+ was Node 6,
        #       use Python 2.7 for anything older.
        python_compat="python3+" if release["version"].major >= 6 else "python2_7",
    )
```

Let's tweak the template to use the newly added variable as well:

```bash
PYTHON_COMPAT=( {{ python_compat }} )
```

And if we try again:

```bash
$ doit
# emerge -a =nodejs-4.9.1
$ node --version
v4.9.1
```

If this isn't cool, then I don't know what is :D

As always, full source can be found
[here](https://code.funtoo.org/bitbucket/users/invakid404/repos/nodejs-overlay/browse).