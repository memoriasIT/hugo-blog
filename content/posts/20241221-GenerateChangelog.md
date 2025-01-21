+++
title = "Generating a changelog from your git commits"
date = "2024-12-21"
author = "memoriasIT"
description = "If you do some kind of software development, you probably are familiar with changelogs. These changelogs are especially useful for other people who don’t work on your project to at a glance know what’s new in X version.\n\nHowever, maintaining this kind of files is always a pain and requires a lot of manual labor. Today, I will show you how we can achieve something similar with only using git."
+++

If you do some kind of software development, you probably are familiar with changelogs. These changelogs are especially useful for other people who don't work on your project to at a glance know what's new in X version.

```markdown
# 8.1.4

- docs: improve diagrams
- chore: update copyright year
- chore: update sponsors

# 8.1.3

- chore: update sponsors
```

However, maintaining this kind of files is always a pain and requires a lot of manual labor. Today, I will show you how we can achieve something similar with only using git.
For that, we will leverage the `git log` command.

# Playing with git log

`git log` is just a command to list your commits, however, it has an option called `pretty` that allows you to print the output with a certain format.
The full documentation can be seen [here](https://git-scm.com/docs/pretty-formats), but I will quickly go over the custom formats and simplify it for you with examples.

The main idea is to use placeholders. For example:

- `%H` : commit hash
- `%an` : author name
- `%b` : body

With that in mind, we can just clone a repository (with some depth to get the git history) and run Git log:

```text
❯ git clone https://github.com/felangel/bloc.git --depth 20
❯ git log --pretty='format:%s | %an - %h' | tail -n 5

docs: adjust sponsor grid columns (#4304) | Felix Angelov - 5ca35a7
chore(bloc_lint): remove parabeac from sponsors grid | Felix Angelov - a3aed13
ci: upgrade `pana` to `0.22.17` (#4303) | Felix Angelov - 5bd7404
chore: remove parabeac from sponsors (#4302) | Felix Angelov - 92adcb5
chore(deps): bump astro from 4.16.9 to 4.16.18 in /docs (#4301) | dependabot[bot] - 54c224a
```

As you can see, this is already pretty nice; especially because it is using [conventional commits](https://www.conventionalcommits.org/en/v1.0.0/).

To push this further, we can also play with tags. For example, we might only want to get the changelog since the last tag (only the new commits).

```text
❯ git log $(git describe --tags --abbrev=0)..HEAD --pretty='format:%s | %an - %h'
```

Note how in the example above I did not have to specify the tag name (I use the last one), but if you wanted, all you had to do is to specify the tag or commit.

```text
❯ git log a3aed13..HEAD --pretty='format:%s | %an - %h'

docs: adjust sponsor grid columns (#4304) | Felix Angelov - 5ca35a7
chore(bloc_lint): remove parabeac from sponsors grid | Felix Angelov - a3aed13
```

# Conclusion

Creating changelogs is always boring, but just like every boring thing in IT, we can automate it!

We just saw that just by using our git logs, we can generate beautiful output to use in our changelog.
And obviously, this can even be run in your release workflows so zero manual labor is needed!

As always, feel free to send me a message!
I would be very happy to hear your thoughts :)
