+++
title = "My PR check pipeline"
date = "2024-10-29"
author = "memoriasIT"
description = "As someone who works in a consultancy firm, I create many projects. This process can be tedious and repetitive. That's why creating \"building blocks\" can be really helpful..."
+++

As someone who works in a consultancy firm, I create many projects. This process can be tedious and repetitive. That's why creating "building blocks" can be really helpful.

I believe this is specially true for the CI/CD of the applications.
In today's post, I would like to create the (my) "perfect" PR check pipeline for your github repo.

As a Flutter developer, I often find myself inspired by the work of VeryGoodVentures. Not only is it of great quality, but it is also MIT licensed. If you are familiar with [VeryGoodWorkflows](https://www.github.com/VeryGoodOpenSource/very_good_workflows/tree/main), this might feel familiar.

# Let's get to the point!

Let's first make a list of all the requirements I would like my action to have:

- **Simple:** Understanding the actions should not require a lot of knowledge.
- **Reusable:** There might be small differences between projects (architecture, versions, requirements, etc.). The actions should be flexible enough to account for that.

## What do I specifically want my PR to do?

- **Analysis:** Either with dart static analysis or something like SonarCube. For simplicity I will use the default dart analysis. But I will probably tackle the implementation of SonarCube in the near future (I am intrigued by their _"new"_ dart support).

- **Vulnerability Scanner:** A couple of months ago, I discovered [OSV-scanner](https://security.googleblog.com/2022/12/announcing-osv-scanner-vulnerability.html). This scanner, made by Google, aggregates multiple vulnerability databases to find existing vulnerabilities affecting your project's dependencies. Pub.dev is included, so that's awesome for me! But if you do JS, Go, Rust, Python... it will be great for you too!

- **PR format:** I would like to make sure that the PR titles follow the same style for future search/reference. One common style to use is the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) spec.

- **Tests:** Testing is can be a great safety net for errors sometimes. Let's run tests and add support for minimum coverage required.

- **Build:** I feel like this could spark controversy, as a build can take considerable time. But, personally I would like to have the assurance that what is in develop/master can be built successfully.

## The full workflow

```yaml
name: pull-request-check

# Cancel the active runs of the same branch of the same workflow (run only one
# at the same time)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

on:
  workflow_dispatch:

  # Run workflow if pull request targets main/develop, master or a release branch.
  branches:
    - main
    - develop
    - master
    - "release/**"

  # It will run when the PR is opened, close & reopened, set from draft to ready
  # to review, the title/body/base is changed or the head branch was updated.
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize, edited]

# Only read permission to contents and pull-requests is required, if you decide
# to expand on this workflow you might need to add more permissions
permissions:
  pull-requests: read
  contents: read

jobs:
 semanticTitle:
    # By tagging the runner we can use multiple runners according to if we are
    # building for iOS (that requires Mac OS) or not. In this case, we don't care.
    runs-on: [android, ios]

    steps:
        # feat(MIT-33): Add `Button` component
        # ^    ^    ^
        # |    |    |__ Subject
        # |    |_______ Scope
        # |____________ Type
      - name: 🤖 Ensure Commit is Semantic
        uses: amannn/action-semantic-pull-request@0723387faaf9b38adef4775cd42cfd5155ed6017 # v5.5.3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          # If PR is only a single commit we also check for the valid pattern
          validateSingleCommit: true
          # Scopes can be passed from the project itself. This will allow to
          # have custom JIRA ticket numbers. For example: "MIT-33"
          scopes: ${{inputs.scopes}}
          # If for some reason we want to skip the check in a PR we can add the
          # label "ignore-semantic-pull-request" to it to skip the check.
          ignoreLabels: |
            ignore-semantic-pull-request

  test:
    steps:
      - run: echo Hello world!
# TODO JOB SUMMARY
# https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#adding-a-job-summary
```
