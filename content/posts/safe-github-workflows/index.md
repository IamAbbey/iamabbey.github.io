---
title: 'Are your Github Workflows safe from attack?'
date: 2025-03-20T12:05:08+01:00
# weight: 1
# aliases: ["/first"]
tags: ["github", "workflow", "ci"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
summary: "This article explain common problem with Github workflow and recommends solution."
# description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/IamAbbey/iamabbey.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Github is the world's largest source code host, having over 100 million developers and more than 420 million repositories.

One of the most utilized feature of Github is [Github Actions](https://github.com/features/actions). GitHub Actions makes it easy to automate workflows, offering one of the best Continuous Integration and Continuous Deployment (CI/CD).

If you have worked on any software as a solo developer, for a company or contributed to open source making using of Github, you have most likely used github actions. If this is your first time hearing about Github Actions, [read more about it here](https://docs.github.com/en/actions/writing-workflows/quickstart).

Github actions can be shared publicly for others people to make use of. There are thousands of shared public actions which offers different and useful functionalities, so you do not have to write them yourself.

Example of GitHub Actions available are:

- [Build and push Docker images](https://github.com/marketplace/actions/build-and-push-docker-images): build and push Docker images with BuildKit.
- [Docker Login](https://github.com/marketplace/actions/docker-login): sign in to a Docker registry.
- [actions/setup-python](https://github.com/marketplace/actions/setup-python): to have python setup within your workflow run
- [actions/checkout](https://github.com/marketplace/actions/checkout): to checks-out your repository under $GITHUB_WORKSPACE, so your workflow can access it.

Here comes the problem!

Workflow users may reference this available actions in multiple ways

#### Using release tags
```yaml
steps:
    - uses: actions/checkout@v4 # using release tags
```

#### Using branch name
```yaml
steps:
    - uses: actions/checkout@master # using branch name
```

#### Using commit sha
```yaml
steps:
    - uses: actions/javascript-action@a824008085750b8e136effc585c3cd6082bd575f # using commit sha
```

Have you seen the problem? 
![bad actors](/posts/safe-github-workflows/images/9o0i3g.jpg)

A lot of trending actions found at the marketplace are using secrets to perform their tasks. Some may require username, password or access token for different functionalities, like authentication, to able to use some services.

For example, in using the above referenced action `actions/docker-login`

```yaml
name: ci

on:
  push:
    branches: main

jobs:
  login:
    runs-on: ubuntu-latest
    steps:
      -
        name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

I doubt if users of Github Actions go through in details the source code of this kind of actions to see how their credentials is being used.

We in most times trust this actions creator, what if the the publisher we trust decides to let someone else support this code (thats Open Source thing right? Collaboration!) ?

Any codebase maintainer can update branch or tags, THIS IS THE PROBLEM

Look at this scenario

- Maintainer A (with good intent) pushes code to a branch (often main or master) or with a tag (has commit abc123)
- Maintainer B (with bad intent) makes code changes update the branch or tag with his code changes (new commit def456)

> Each Git commit receives a calculated SHA value, which is unique and immutable. 

> Git tags can be deleted or moved

WE HAVE A PROBLEM in our workflow now
```yaml
steps:
    - uses: actions/some-action@<branch_name> # using branch
```
When we trigger our workflow, our workflow runs will use updated code from the action's branch (NOW OUR WORKFLOW will work with THE BAD CODE)
```yaml
steps:
    - uses: actions/some-action@v<tag> # using tag reference
```
Just like the above, if we use tags, further updates to this actions will be used during our workflow run

USING FULL COMMIT SHA IS THE ONLY WAY TO BE SAFE
```yaml
steps:
    - uses: actions/some-action@<commit_sha> # using tag reference
```
Using commit sha, means every time our workflow runs we make use of the same action code.

> The caveat to using commit SHA is that our workflow will NOT have access to updated action code.
> We might have to manually update this SHA to latest versions if we really want to use an updated action code functionality.

Using our scenario, when we use `actions/some-action@abc123` we are SAFE from the bad code vulnerabilities introduce by the new commit `def456`.


### Is this even a real problem to be bothered about? 

Well, if anything, we should learn from history, 

- At around 4:00 PM March 14th, 2025 UTC, a popular GitHub Action (tj-actions/changed-files) was fully compromised. Someone committed a base64-encoded payload that runs a script that in turn prints out encoded secrets [read about the incident](https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised).

![pleading](/posts/safe-github-workflows/images/please-use.png)

### Useful links

- https://julienrenaux.fr/2019/12/20/github-actions-security-risk/
- https://docs.github.com/en/actions/sharing-automations/creating-actions/about-custom-actions