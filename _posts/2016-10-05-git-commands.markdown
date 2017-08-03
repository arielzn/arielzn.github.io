---
layout: post
title:  "Useful git related commands"
date:   2016-10-05 02:06:07 +0200
categories: git
---

## How to keep a fork synced with upstream


Assuming the repo 
```
https://github.com/someuser/repo.git
```
was forked in your github account to 
```
https://github.com/youruser/repo.git
```

Clone your fork and add upstream repository

```bash
$ git clone  https://github.com/youruser/repo.git
$ git remote add upstream https://github.com/otheruser/repo.git
```

If the process went smooth you should get

```bash
$ git remote -v

origin    https://github.com/youruser/repo.git (fetch)
origin    https://github.com/youruser/repo.git (push)
upstream  https://github.com/someuser/repo.git (fetch)
upstream  https://github.com/someuser/repo.git (push)
```


When commits are made to the upstream repo you'll see on github when you are standing on the updated branch
"this branch is XXX commits behind someuser:updatedbranch".

To sync, go to your local repo, fetch the upstream commits and merge with the local branch. If the updated
branch is e.g. the master one, the steps would be

```bash
$ git fetch upstream
$ git checkout master
$ git merge upstream/master
```

and then push to your github fork to update it

```bash
$ git push master
```

That's it.


When developing your fork it's a good idea to work in an independent 'dev' branch,
this way the master can be kept in sync with upstream (by following the previous steps) and 
compared with our current work one.
We create and push it doing

```bash
$ git checkout -b dev
$ git push origin dev
```


## How to use diferent identities for git repos

Option available starting with git v2.8 [(relase entry)](https://github.com/blog/2131-git-2-8-has-been-released).

Tell Git to force you set user.name and user.email explicitly before it will let you commit:

```bash
$ git config --global user.useconfigonly true
```

In case you had previously configured a global identitiy remove that

```bash
$ git config --global --unset-all user.email
```

Afterwards, you will have to set your specific identity for every repository you had
(and every new one you clone).
To set the desired one for each repo do

```bash
$ git config user.email "you@example.com"
$ git config user.name "Your Name"
```

In case you forget to setup that for one particular repo you'll be reminded to do so in your next commit.


