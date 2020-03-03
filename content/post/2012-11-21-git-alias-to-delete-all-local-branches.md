---
title: Git alias to delete all local branches
author: Pedro
type: post
date: 2012-11-21T18:11:34+00:00
aliases: 
  - /2012/11/21/git-alias-to-delete-all-local-branches/

dsq_thread_id:
  - 938075566
categories:
  - Tips
tags:
  - git
  - tip

---
In my current project we have to use TFS as our _remote_ repo. Locally I use [git-tfs][1] so that I can still be productive and do, ya know, work. [Jimmy has a post describing in details the workflow that we use here][2] but the TL;DR: version is:  All work is done on local topic branches; You push to TFS from the topic branch; TFS, after running the build/tests, will commit the changes that were pushed; You pull the commited changes to master. You create a new topic branch, rinse and repeat.

I've been working with this workflow for more than an year and it's working great. It has one side effect, though. It can leave tons of non-merged branches, if you don't delete the topic branch after you pushed. So, once in a while it is time for some branching cleanup.

At first I was doing the cleanup manually, but I'm really lazy and I'd rather tell the computer to do the work for me. So, here is my git alias to  delete all local topic branches except `master`:

```bash
dab = !git checkout master && git branch | grep -v "master" | xargs git branch -D
```

Now, every time I want to do some branch cleanup I simply do `git dab` and all the junk branches are gone.


 [1]: https://github.com/git-tfs/git-tfs
 [2]: http://lostechies.com/jimmybogard/2011/09/20/git-workflows-with-git-tfs/
