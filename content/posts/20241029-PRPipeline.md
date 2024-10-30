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

- **Vulnerability Scanner:** A couple of months ago, I discovered [OSV-scanner](https://security.googleblog.com/2022/12/announcing-osv-scanner-vulnerability.html). This scanner, made by Google, aggregates multiple vulnerability databases to find existing vulnerabilities affecting your project's dependencies. Pub.dev is included, so that's awesome for me! But if you do JS, Go, Rust, Python... it will be great for you too!

- **PR format:** I would like to make sure that the PR titles follow the same style for future search/reference. One common style to use is the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) spec.

- **Analysis:** Either with dart static analysis or something like SonarCube. For simplicity I will use the default dart analysis. But I will probably tackle the implementation of SonarCube in the near future (I am intrigued by their _"new"_ dart support).

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
    inputs:
      # Custom scopes for the semanticTitle job
      scopes:
        required: false
        type: string
      # Directories to analyze with flutter analyze
      analyze_directories:
        required: false
        type: string
        default: "lib test"
      # In multipackage projects the build directory might be different
      working_directory:
        required: false
        type: string
        default: "."
      # Pre-run hook, useful to run custom commands before the build (e.g. melos get)
      setup:
        required: false
        type: string
        default: ""
      # After-run hook, useful to run custom commands after the build
      cleanUp:
        required: false
        type: string
        default: ""
      # Git clone fetch-depth
      fetchDepth:
        required: false
        type: string
        default: "50"


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
    # This is a great way to save costs if you are not self-hosting.
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


  vulnerability-checker:
    name: 🦠️Vulnerability check
    runs-on: [android, ios]
    steps:
        # Sets the SSH key to clone the repo
      - name: 🔑Setup repo SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_KEY }}
          known_hosts: ${{ secrets.SSH_KNOWN_HOSTS }}
          if_key_exists: ignore

        # Checks out the code repo
      - name: 📚Checkout code
        uses: actions/checkout@v3
        with:
          ref: ${{ github.head_ref }}
          fetch-depth: 0

        # Uses OSV-Scanner to find known vulnerabilities in packages
        # This is not default in github runners (use brew or snap to get it)
      - name: 🦠️ Vulnerability Check
        run: |
          osv-scanner -lockfile=./pubspec.lock

  build:
    defaults:
      run:
        working-directory: ${{inputs.working_directory}}

    runs-on: [android]

    steps:
      - name: 🔑Setup repo SSH key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_KEY }}
          known_hosts: ${{ secrets.SSH_KNOWN_HOSTS }}
          if_key_exists: ignore

      - name: 📚 Git Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
          # By default only the last commit is fetched, but we will generate a
          # changelog automatically, so this will be necessary.
          fetch-depth: ${{inputs.fetchDepth}}

      - name: ⬇️Install Flutter version used in the project
        # I use FVM, which is the flutter equivalent of NPM for JS or RVM for ruby.
        # It uses a config file in the project to specify the version (.fvmrc).
        run: fvm install

      - name: 🤫 Set SSH Key
        env:
          ssh_key: ${{secrets.ssh_key}}
        if: env.ssh_key != null
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{secrets.ssh_key}}

      - name: ⬇️Get Flutter dependencies
        run: fvm flutter pub get

      - name: ⚠️Analyze for lint errors/warnings
        run: fvm flutter analyze

      - name: 🧪Run tests
        run: fvm flutter test --no-pub --coverage

      - name: 📊 Check Code Coverage
        uses: VeryGoodOpenSource/very_good_coverage@v3
        with:
          path: ${{inputs.working_directory}}/coverage/lcov.info
          exclude: ${{inputs.coverage_excludes}}
          min_coverage: ${{inputs.min_coverage}}

      - name: ⬇Install bundle dependencies in Android
        working-directory: ${{inputs.working_directory}}/android
        run: bundle install

      - name: ⚙️Build APK
        working-directory: ${{inputs.working_directory}}/android
        run: fvm flutter build apk

# TODO JOB SUMMARY
# https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#adding-a-job-summary
```
