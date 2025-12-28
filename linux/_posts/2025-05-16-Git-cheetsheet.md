---
layout: post
title: "GIT cheatsheet"
categories: [linux, devops]
tags: [git, cheatsheet]
---

This is a simple cheat sheet of common git commands that might be useful.

### Create local branch
Create and switch to a new local branch
```bash
git checkout -b <branch_name>
```

### Push local branch to remote
Push your local branch to the remote repository
```bash
git push origin <branch_name>  
```

### Remove local branch
Delete a local branch safely (only if merged)
```bash
git branch -d <branch_name>
```

### Remove remote branch
Delete a branch from the remote repository
```bash
git push -d <remote_name> <branch_name>
```

### Switch to remote branch
Switch to an existing branch
```bash
git switch <remote_name>
```

or on older GIT clients create and track a remote branch locally

```bash
git checkout -b <branch_name> <name of remote>/<branch_name>
```

### Stash changes
Save your local modifications temporarily
```bash
git stash
```

### Make a branch for local changes and push to remote
```bash
git checkout -b <branch_name>
git add .
git commit -m "your commit message"
git push origin <branch_name>
```
