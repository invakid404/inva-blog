---
id: CHZJs9AYG3tpebkEwSBpU
title: Automatic Publishing with Remote Vaults
desc: ''
updated: 1642972165628
created: 1642969203545
excerpt: >-
  How I publish my multi-vault Dendron workspace.
---

![vaults](/assets/images/vaults.jpg)

I have a single Dendron workspace with multiple remote vaults connected to it.
It lives in [this](https://github.com/invakid404/dendron-workspace) repo and has
both my public vaults (like this blog) and my private vaults (personal notes).

The pros of such a setup are that I only have a single instance of VS Code
dedicated to Dendron. Also, I like having a single point of access for all of my
notes since it minimizes the shuffling around I need to do.

The biggest con was that it was annoying to publish it via a CI/CD pipeline
since some of the vaults live in private repos. The reason this is a problem is
that `dendron workspace init` tries to clone those vaults but fails.

At first, I hacked together a couple of [yq](https://github.com/kislyuk/yq)
commands to remove all vaults with `private` visibility, but those are too ugly
to show in this post :D.

Instead, I decided to put together a custom action for GitHub Actions, which
does the same thing for me in a tidier way and a bit more. I decided to call it
"Dendron Multi-Publish", and you can find it
[here](https://github.com/invakid404/dendron-multi-publish-action).

The action does a few things. It:
- Initializes the workspace - pulls in all of your remote workspaces and vaults.
- Checks whether there are any changes - it hashes all of your notes and assets
  to figure out whether anything is modified.
- Exports your notes for publishing in GitHub Pages.
- Caches the published notes - it uses GH Actions cache to avoid having to
  re-export the same notes.
  
You can check out the way I'm using it
[here](https://github.com/invakid404/dendron-workspace/blob/master/.github/workflows/publish.yml).