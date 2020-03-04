---
title: "Publishing a Hugo Site to GitHub Pages via Github Actions"
date: 2020-03-03T12:00:00-05:00
comments: false
tags: ["hugo"]
---

## Background

As of this writing, I am currently using the [Hugo](https://gohugo.io/) static site generator for this site and publishing to [GitHub pages](https://pages.github.com/). [The code](https://github.com/zporter/zachporter.dev/) is open-source on GitHub. To organize the code, I like to keep the Hugo source files in the `master` branch and publish from the [`gh-pages` branch](https://github.com/zporter/zachporter.dev/tree/gh-pages). To publish the site, I use a script that I can run locally, but I wanted to check out the relatively new [GitHub Actions](https://github.com/features/actions) to see if I could automate the process after a push to the `master` branch. This article will cover the journey to GitHub Actions along with my takeaways.

## Setup

Getting started with GitHub Actions is pretty straightforward. GitHub provides a great starter template and has some examples in their documentation. There's also [a marketplace](https://github.com/marketplace?type=actions) for actions to use or, in my case, use as inspiration to write my own. One downside to the marketplace is that there seems to be quite a bit of legacy code that I can only assume used earlier versions of GitHub Actions. As far as I can tell, GitHub has since changed to using a YAML-formatted file and have changed how variables can be exposed to the workspace as environment variables. The source code of the actions in the Marketplace is pretty easy to read, so you can make a judgement call as to whether you can pull it in or whether you need to roll your own.

## Implementation

I started with a workflow titled "Hugo Build and Release" with the configuration and steps stored in `.github/workflows/hugo_build_and_release.yml`. The first line in the file is the name of the workflow.

```yaml
name: Hugo Build and Release
```

Simple enough. The next section in the workflow deals with the events that need to happen on GitHub in order to trigger the Action to run.

```yaml
on:
  push:
    branches:
    - master
```

I knew I wanted to trigger the publish action on push to `master`, so this is the configuration that I chose. You can review [the documentation](https://help.github.com/en/articles/workflow-syntax-for-github-actions#on) for more actions to use to trigger the workflow.

Next up in the file is the `jobs` specification.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
```

Here, I only have one job titled `deploy`. This is because by default `jobs` run in parallel. My workflow is very sequential:

1. Checkout the source code
2. Build the Hugo site
3. Push the static site to the `gh-pages` branch

It made sense to only define one job to execute a sequence of steps that are dependent on each other. If you have a workflow that can easily run in parallel, then I encourage you to consult [the documentation](https://help.github.com/en/articles/workflow-syntax-for-github-actions#jobs) on how best to configure.

I chose to run my job on the `ubuntu-latest` virtual environment, since that's the one I'm most familiar with. GitHub supports a number of other [virtual environments](https://help.github.com/en/articles/virtual-environments-for-github-actions#supported-virtual-environments-and-hardware-resources). Choose the one that best fits your workflow.

Now that I have a job, I need to define some steps.

```yaml
steps:
- uses: actions/checkout@v1
```

GitHub gave me a `checkout` action as part of the starter template. This will checkout the source code in the `master` branch and store it in a workspace on the virtual environment.

## Problems with Checkout

At the time, I had two [git submodules](https://github.com/blog/2104-working-with-submodules) in the project: one for [the theme](https://github.com/zporter/hugo-theme-hello-friend) and one for the `public` directory, which was pointed to the `gh-pages` branch for publishing the GitHub Pages site. I've since moved away from the second submodule and followed [Hugo's documentation](https://gohugo.io/hosting-and-deployment/hosting-on-github/#deployment-of-project-pages-from-your-gh-pages-branch) on deploying from the `master` branch to the remote `gh-pages` branch on GitHub. To pull in the git submodule for the theme, I had to add a simple option to allow checking out submodules:

```yaml
- uses: actions/checkout@v1
  with:
    submodules: true
```

Running the action so far resulted in a success, so now it was time to move onto building the static site.

## Installing Hugo

I knew I would need Hugo in order to build the static site. It was just a matter of determining the best way to install the dependency. After reviewing several existing Actions in the Marketplace, I settled on adding the following step to my job:

```yaml
- name: Install Dependencies
  env:
    HUGO_VERSION: 0.58.3
  run: |
    echo "installing hugo $HUGO_VERSION"
    curl -sL -o /tmp/hugo.deb \
    https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.deb && \
    sudo dpkg -i /tmp/hugo.deb && \
    rm /tmp/hugo.deb
```

Since this is in an Ubuntu container, [the Hugo Debian package](https://gohugo.io/getting-started/installing/#install-hugo-from-tarball) can be downloaded with the version of Hugo I need.

With the Hugo dependency installed, it's now time to publish the site.

## Publishing

I could have placed the publishing commands in the Actions workflow file, but I had already been using a script to publish the site, and I would like to maintain the ability to publish outside of the GitHub Action. I had to make some modifications to my previous publishing workflow, but I think the final version ended up a bit cleaner:

```sh
# ./bin/publish.sh

#!/bin/sh

if [ "`git status -s`" ]
then
    echo "The working directory is dirty. Please commit any pending changes."
    exit 1;
fi

echo "Deleting old publication"
rm -rf public
mkdir public
git worktree prune
rm -rf .git/worktrees/public/

echo "Checking out gh-pages branch into public"
git worktree add -B gh-pages public origin/gh-pages

echo "Removing existing files"
rm -rf public/*

echo "Generating site"
hugo

echo "Updating gh-pages branch"
cd public && git add --all && git commit -m "Publishing to gh-pages (publish.sh)"

echo "Pushing to github"
git push -f ${REPOSITORY:-origin} gh-pages
```

Each of the `echo` commands above should make it pretty clear what is happening. The script is leveraging [Git worktrees](https://git-scm.com/docs/git-worktree/), which comes in handy when committing and pushing work in another directory to a remote branch. In this case, the site code gets generated into the `public/` directory, and the changes are committed and pushed up to the remote `gh-pages` branch. GitHub takes over from there to post the content here.

To publish from my computer, I now only need to specify the `REPOSITORY` environment variable, and I'm up and running.

To run this from the GitHub Action, I just add the following to my `hugo_build_and_release.yml` specification:

```yaml
- name: Publish Site
  env:
    REPOSITORY: "https://${{ secrets.GITHUB_PAT }}@github.com/${{ github.repository }}.git"
  run: ./bin/publish.sh
```

This uses [a Personal Access Token](https://github.blog/2013-05-16-personal-api-tokens/), which, at the time of this writing, was required in order to effectively trigger the GitHub Pages publish action. That may be fixed in the future for [the already provided access token](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/authenticating-with-the-github_token#using-the-github_token-in-a-workflow) within the GitHub Action. I gave my Personal Access Token `repo` and `admin:repo_hook` privileges. This might've been overkill, but it was very difficult to dig up information on what exactly was required to trigger a GitHub Pages publication. Once that Personal Access Token was generated, I stored it as a Secret on the repository and referenced it as `secrets.GITHUB_PAT`.

## Conclusion

That concludes the approach that I took to use GitHub Actions to publish my site built using the Hugo static site generator on GitHub Pages. The process was a lot of trial and error. Being early days for GitHub Actions, the documentation was difficult to search and sift through, and having no feedback that my site was failing to build on GitHub Pages after a push was difficult to debug and fix. I ended up reading what others had done on the Actions Marketplace, so kudos to those trailblazers for leaving comments as to why they needed to use a Personal Access Token.

The final workflow file ended up like this:

```yaml
name: Hugo Build and Release

on:
  push:
    branches:
    - master

jobs:
  deploy:

    runs-on: ubuntu-latest
    
    steps:
    - name: Configure Git for Github
      run: |
        git config --global user.name "${GITHUB_ACTOR}"
        git config --global user.email "${GITHUB_ACTOR}@users.noreply.github.com"
    - uses: actions/checkout@v1
      with:
        submodules: true
    - name: Install Dependencies
      env:
        HUGO_VERSION: 0.58.3
      run: |
        echo "installing hugo $HUGO_VERSION"
        curl -sL -o /tmp/hugo.deb \
        https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.deb && \
        sudo dpkg -i /tmp/hugo.deb && \
        rm /tmp/hugo.deb
    - name: Publish Site
      env:
        REPOSITORY: "https://${{ secrets.GITHUB_PAT }}@github.com/${{ github.repository }}.git"
      run: ./bin/publish.sh
```

Overall, I am thankful that this is available to me so I can push changes to the `master` branch and not need to remember a separate publishing step. I just hope that GitHub can continue to improve the documentation and provide feedback when services fail. I encourage you, dear reader, to find ways to automate your workflows with GitHub Actions. Write up your experiences and share back with the community. 💜
