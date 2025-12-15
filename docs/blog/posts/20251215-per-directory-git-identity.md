---
date: 2025-12-15
title: Per directory git user details for different projects
authors:
  - onestepcloser84
categories:
  - General
tags:
  - git
  - config
  - identity
slug: per-directory-git-indentity
---

# Per-directory git user details with includeIf

If you keep repos grouped by folder, for example:
- All work related git repositories in folder `~/code/work`
- All personal git repositories in folder `~/code/personal`

then you can set different user details, like `user.name` or `user.email`, automatically using conditional includes.

## Example

Create `~/.gitconfig` file and ensure `includeIf` for each repository group:

```ini
[user]
  name = <Your Name>
  email = your-primary-email@domain.com

## other options

[includeIf "gitdir:~/code/work/"]
  path = ~/.gitconfig-work

[includeIf "gitdir:~/code/personal/"]
  path = ~/.gitconfig-personal

## rest of the options
```

and then create `~/.gitconfig-work` and `~/.gitconfig-personal` files with their own contents.

`~/.gitconfig-work`:
```ini
[user]
  email = work-email@company.com
```

`~/.gitconfig-personal`:
```ini
[user]
  email = personal-email@example.com
```
