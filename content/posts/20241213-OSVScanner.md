+++
title = "Scanning for vulnerabilities in your dependencies"
date = "2024-12-13"
author = "memoriasIT"
description = ""
+++

A while ago, I was working for a client that had a specific requirement: they wanted to check for the [OWASP top ten security risks](https://owasp.org/www-project-top-ten/).

If you are not aware of what these lists are, OWASP is a well known non-profit organization that provides security related resources. They also make these security reports which take into account their own data and the community's to list the top ten most common security risks of that year.

One of the top vulnerabilities is usually using "Vulnerable and Outdated Components" and this made me wonder if I can automate something to check for known vulnerable dependencies.

Luckily for me, Google already made a great scanner for this that works with many languages (C++, Dart, Go, Java, Python…). Their solution is called [OSVScanner](https://google.github.io/osv-scanner/).

This scanner uses the [OSV Database](https://osv.dev/) as backbone (a vulnerability database for open source projects).

Today, I will show you how you can very easily implement this in your workflows.

# Installing the tool

If you use a package manager, this might be your best bet. For example, for macOS you can just use brew:

```text
brew install osv-scanner
```

However, you can also install from source (you need to Go installed):

```text
go install github.com/google/osv-scanner/cmd/osv-scanner@v1
```

Pre-compiled releases can also be found in their GitHub (inb4 [just give me the freaking EXE](https://www.reddit.com/r/github/comments/1at9br4/i_am_new_to_github_and_i_have_lots_to_say/)):

[https://github.com/google/osv-scanner/releases](https://github.com/google/osv-scanner/releases)

# Usage

Using it is also really easy, just specify a path or lockfile and it will do its magic!

```text
❯ os-scanner -r .
Scanning dir .
Scanning /Users/me/example/ at commit redacted
No issues found
```

If it finds something, it will look like this:

```text
❯ osv-scanner --lockfile 'pubspec.lock'
Scanned /Users/me/example/pubspec.lock file and found 204 packages
╭─────────────────────────────────────┬──────┬───────────┬────────────────────────────┬─────────┬──────────────╮
│ OSV URL                             │ CVSS │ ECOSYSTEM │ PACKAGE                    │ VERSION │ SOURCE       │
├─────────────────────────────────────┼──────┼───────────┼────────────────────────────┼─────────┼──────────────┤
│ https://osv.dev/GHSA-3hpf-ff72-j67p │ 3.0  │ Pub       │ shared_preferences_android │ 2.3.3   │ pubspec.lock │
╰─────────────────────────────────────┴──────┴───────────┴────────────────────────────┴─────────┴──────────────╯
```

The output will either be "No issues found", or a table like the one above.
In the table you will find a link to the OSV database, the common vulnerability score (CVSS), the ecosystem (Go, crates, Pub, etc.), the package name and version and where the dependency was found.

Usually the workarounds and fixes are pretty easy and detailed in the OSV database entry.

# Usage in CI/CD

Google does provide some [reusable PR workflows](https://github.com/google/osv-scanner-action/tree/main/.github/workflows) you can use. There is also [documentation](https://google.github.io/osv-scanner/github-action/) to use them. However, I find them more complex than they should really be.

The program already returns 0 or 1 if it found something, so we can just use the same setup as locally and it will work automatically.
I hereby provide an example job that would do the trick:

1. Checkout the code
2. Set up the Go version
3. Install from source and run

```yaml
vulnerability-checker:
  name: 🦠️Vulnerability check
  runs-on: ubuntu-24.04
  steps:
    - name: 🔑Setup repo SSH key
      uses: shimataro/ssh-key-action@v2
      with:
        key: ${{ secrets.SSH_KEY }}
        known_hosts: ${{ secrets.SSH_KNOWN_HOSTS }}
        if_key_exists: ignore

    - name: 📚Checkout code
      uses: actions/checkout@v3
      with:
        ref: ${{ github.head_ref }}
        fetch-depth: 0

    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: "1.23.3"

    - name: 🦠️ Vulnerability Check
      run: |
        go install github.com/google/osv-scanner/cmd/osv-scanner@v1
        osv-scanner -lockfile=./pubspec.lock
```

If you have the luxury of running on macOS images, you can skip the set up Go step and just use brew as I showed in the [installation](#installing-the-tool) section. And obviously, if you are self-hosting, you don’t need to install it every time.

# Conclusion

We found out that running a vulnerability scanner is one of those low effort, high impact changes to implement.
To be fair, I am not sure why it is not something more popular or talked about, at least in the Flutter world.
Hopefully this blog post is able to convince at least one more reader out in the world to implement it in their system :)

Please let me know your thoughts or concerns, either in the comments or in any of my [social media platforms](https://memoriasit.com/links).
