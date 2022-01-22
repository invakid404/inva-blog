---
id: 4ZLcPdoCZQEpFIZ028gen
title: Automating seed-tldr
desc: ''
updated: 1642862280282
created: 1642850815639
excerpt: >-
  Incremental automatic updates for seed-tldr: where tldr-pages meet Dendron.
---

![seed-tldr](/assets/images/seed-tldr.png)

I recently found out about [Dendron](https://www.dendron.so) through the Funtoo
Linux Discord server and thought it was a very cool project. In the process of
experimenting with it, I was pleasantly surprised to see that I could add
[tldr-pages](https://github.com/tldr-pages/tldr) to my workspace. 

As a person who uses tldr-pages daily (I tend to forget how to do basic stuff in
the command line quite often), I immediately liked the idea of using Dendron to
browse them instead of a standalone tldr client. That way, I can easily search
through my cheatsheets and tldr-pages at once. I added it to my Dendron
workspace, tried to look up the page for
[ego](https://www.funtoo.org/Package:Ego), but it wasn't there... Hmm, that's
weird, I'm sure there's a page for ego since I've contributed it myself...

I figured out that the tldr pages for Dendron come from the [seed-tldr
repo](https://github.com/kevinslin/seed-tldr), and it struck me as a little bit
odd that the repo was last updated in June 2021. I would've thought such things
would be automatically kept up-to-date with some CI/CD pipeline, but that wasn't
the case. I'm a big fan of automating stuff, so I thought this was a perfect
opportunity to contribute!

My idea was to add the tldr-pages repository as a submodule of seed-tldr and use
it to determine what has changed since the last update. There already was a
GitHub Actions workflow, but it only published the imported pages on push to
main. Thankfully, GitHub Actions has a
[schedule](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)
workflow trigger that runs the workflow automatically based on a cron
expression. My first step was to make the workflow run every hour, which I did
like so:
```yml
on:
  push:
    branches:
    - main
  # Run every hour
  schedule:
    - cron: "0 * * * *"
```

Then, I had to come up with a list of files that were changed to determine
whether we need to perform the import or not. First, I had to get the
last/cached commit of the tldr-pages submodule. This can be achieved by running
`git submodule status --cached`. The output of that command looks something like
this:
```
-47445dd7860917026b4df1845f4c54a0f3d6ab94 repos/tldr
```
To extract the actual commit SHA, I piped it into a couple of `cut` commands and
ended up with `git submodule status --cached | grep tldr | cut -d' ' -f1 | cut
-c2-`.

Now, I can update the submodules by running `git submodule update --init
--recursive --remote` and then look at HEAD to determine the latest commit: `cd
repos/tldr && git rev-parse HEAD`.

Armed with this information, I can finally retrieve the list of changed files by
running `git diff --name-only` on the old and new commits. Then, I have to
filter that list of files, since the pod only imports files from the `pages/`
directory. This was easy enough to do with grep: `grep '^pages/'`.

There is a tiny catch, though: grep returns a non-zero exit code if it matches
nothing, and we don't want that. A simple hack to "swallow" the exit code of
grep is to pipe its output into a command which doesn't error out, such as cat:
`grep '^pages/' | cat`.

Now we have the list of changed pages. Great! This means we can use that list to
check whether anything we care about has changed and run the import and publish
scripts. The final catch in this entire process is that running `importPod` does
a full reimport, which results in bogus changes in the `updated` and `created`
frontmatter fields of all pages, and we don't want that.

We can easily fix this issue though. Remember, we have the list of changed files
already, and we can use it to determine which files should have actually changed
after the import. We just need to tweak thinks a bit to match the output file
names. I achieved this with a quick and dirty sed command: `sed -r
's|pages\/(.+?)\/(.+?)\.md|vault\/\1.\2.md|g'`.

To understand what the sed command does, let's look at a particular example.
Let's say that the list of changed files looks like this:
```
pages/common/git-secret.md
pages/linux/ego.md
```

By looking at the `vault/` directory in seed-tldr, we can figure out how Dendron
maps those names:

```
vault/common.git-secret.md
vault/linux.ego.md
```

Basically, the directory name is concatenated to the file name. This is exactly
what the sed command does: it matches the directory name and the file name and
joins them with a dot, also replacing the `pages/` prefix with `vault/`.

As a final step of this process, we can concatenate the result of all the
previous commands on a single line, so that we can pass it to an action that
commits and pushes the changes to main. This is easily achievable with `xargs`. 

This is how we end up with all of these steps of the workflow:
```yml
    - name: Retrieve cached commit SHA for tldr
      id: tldr-cached
      run: |
        echo "::set-output name=SHA::"\
        "$(git submodule status --cached |
          grep tldr |
          cut -d' ' -f1 |
          cut -c2-)"

    - name: Checkout & update submodules
      run: git submodule update --init --recursive --remote

    - name: Retrieve updated commit SHA for tldr
      id: tldr-updated
      run: |
        echo "::set-output name=SHA::$(cd repos/tldr && git rev-parse HEAD)"

    - name: Retrieve list of updated tldr pages
      id: updated-pages
      run: |
        FILES=$(cd repos/tldr && git diff --name-only "${OLD_SHA}" "${NEW_SHA}")
        PAGE_FILES=$(echo "${FILES}" | grep '^pages/' | cat)
        echo "Changed files:"
        echo "${PAGE_FILES}"
        VAULT_FILES=$(echo "${PAGE_FILES}" |
          sed -r 's|pages\/(.+?)\/(.+?)\.md|vault\/\1.\2.md|g')
        echo "::set-output name=FILES::$(echo ${VAULT_FILES} | xargs)"
      env:
        OLD_SHA: ${{ steps.tldr-cached.outputs.SHA }}
        NEW_SHA: ${{ steps.tldr-updated.outputs.SHA }}
```
(_Note: things are broken into multiple lines to avoid horizontal scrolling_)

The rest is easy: we just need to run Dendron to import and publish the pages.
After importing, we also need to push the changed pages and the tldr submodule
to main, which is done like this:
```yml
    - name: Push changes
      uses: EndBug/add-and-commit@v7.0.0
      if: steps.updated-pages.outputs.FILES != ''
      with:
        add: "./repos ${{ steps.updated-pages.outputs.FILES }}"
        message: "autoupdate docs"
        author_name: 'github-actions[bot]'
        author_email: 'github-actions[bot]@users.noreply.github.com'
```

And we're basically done :)