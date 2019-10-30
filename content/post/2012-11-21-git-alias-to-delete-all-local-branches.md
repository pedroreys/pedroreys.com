In my current project we have to use TFS as our &#8220;remote&#8221; repo. Locally I use [git-tfs](https://github.com/git-tfs/git-tfs) so that I can still be productive and do, ya know, work. [Jimmy has a post describing in details the workflow that we use here](http://lostechies.com/jimmybogard/2011/09/20/git-workflows-with-git-tfs/) but the TL;DR: version is:  All work is done on local topic branches; You push to TFS from the topic branch; TFS, after running the build/tests, will commit the changes that were pushed; You pull the commited changes to master. You create a new topic branch, rinse and repeat.

I&#8217;ve been working with this workflow for more than an year and it&#8217;s working great. It has one side effect, though. It can leave tons of non-merged branches, if you don&#8217;t delete the topic branch after you pushed. So, once in a while it is time for some branching cleanup.

At first I was doing the cleanup manually, but I&#8217;m really lazy and I&#8217;d rather tell the computer to do the work for me. So, here is my git alias to  delete all local topic branches except &#8220;master&#8221;:

`dab = !git checkout master && git branch | grep -v "master" | xargs git branch -D`

Now, every time I want to do some branch cleanup I simply do `git dab` and all the junk branches are gone.

&nbsp;