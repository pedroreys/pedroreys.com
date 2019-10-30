---
title: HotFix for Checkdisk hanging at 1 sec
author: Pedro
type: post
date: 2010-07-30T03:22:07+00:00
url: "2010/07/30/hotfix-for-checkdisk-hanging-at-1-sec"
dsq_thread_id:
  - 417083811
categories:
  - Tips
tags:
  - CheckDisk
  - HotFix
  - KB

---
I have a Dell Studio XPS 1640 running Windows 7 Home Premium x64. 

Yesterday it crashed (sigh) and when I rebooted the check disk was triggered. To my, not pleasant, surprise, Check Disk hanged at 1 sec left on the countdown for pressing a key to skip the disk check. Doing a hard reboot several times I was able to start windows again.

Well, after some googling I found the [HOTFIX KB 975778][1] that solved the issue like a charm for me.

 [1]: http://support.microsoft.com/kb/975778
