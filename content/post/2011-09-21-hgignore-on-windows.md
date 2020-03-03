---
title: .hgignore on Windows
author: Pedro
type: post
date: 2011-09-21T17:23:37+00:00
draft: true
aliases:
  - /2011/09/21/hgIgnore-on-windows
categories:
  - Tips
tags:
  - mercurial encoding .hgignore

---
Mercurial has a .hgignore file where you can specify patterns for files that you want for mercurial to ignore. It is similar to .gitignore and there is nothing fancy here.

&#160;

Today I was creating a mercurial repository to store my dot files on Windows. .hgrc, .gitconfig, .bash_profile and so on.

&#160;

So I created a hg repository on my user folder and, of course I wanted to just track some feel files and ignore all the rest. This can be accomplished pretty easily telling Mercurial to ignore everything.

If you do that, only the files that you explicit add are going to be seen by Mercurial, which is precisely what I wanted.

<a href="http://mercurial.selenic.com/wiki/FAQ#FAQ.2BAC8-CommonProblems.I.27d_like_to_put_only_some_few_files_of_a_large_directory_tree_.28home_dir_for_instance.29_under_Mercurial.27s_control.2C_and_it_is_taking_forever_to_diff_or_commit" target="_blank">There is even a FAQ entry explaining how to do that.</a>

So, if you create a .hgignore as seen below is enough to do the trick:

syntax: glob

*

So, I did exactly that “syntax:glob” > .hgignore to create the file and added the “*” to ignore everything.

&#160;

And, it didn’t work.

### 

The Devil Hides in the Details

Opening the .hgignore file in Notepad++ I noted that the file was encoded in “ECS-2 Little Endian”. 

Turns out <a href="http://www.selenic.com/pipermail/mercurial/2008-April/018396.html" target="_blank">hg can’t parse the .hgignore file correctly unless it’s ANSI or UTF-8 encoded</a>, so I changed the encoding to ANSI and voilà, it worked.
