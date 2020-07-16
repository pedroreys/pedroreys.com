---
title: "npm scripts, Windows and Environment Variables - a hairy, hairy yak"
author: Pedro
date: 2020-07-16T00:00:00Z

tags:
  - bugs
  - npm
  - troubleshooting
---

The other day a Headspring colleague asked for help with an weird issue he was running setting up a Github Actions Workflow for [Totem](https://github.com/headspringlabs/totem) and he was getting this error when trying to run the app from within a npm script:


```text
Start-Process : Item has already been added. Key in dictionary: 'NPM_CONFIG_CACHE'  Key being added: 'npm_config_cache'
At line:1 char:1
+ Start-Process 'dotnet' -nnw -argumentlist 'run -p ../Totem/Totem.cspr ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [Start-Process], ArgumentException
    + FullyQualifiedErrorId : System.ArgumentException,Microsoft.PowerShell.Commands.StartProcessCommand
```

In the `project.json` file he had a npm script defined as:

```json
{ 
    "scripts" : {
        "runtestps": "@powershell Start-Process 'dotnet' -nnw -argumentlist 'run -p ../Totem/Totem.csproj --launch-profile \"Testing\" -v diag'"
    }
}
```

When he asked for help to troubleshoot this I got immediately nerd snipped to not only find a workaround but to actually get to the root cause of what was causing it.

## Troubleshooting 

There isn't a right or wrong way to troubleshoot something and different people have different ways they prefer to do it. The steps that I usually take are roughly:

1. Make sure I can reproduce the issue with almost to no changes.
2. Poke around to get myself familiarized with the context, tools, all variables involved, usually bumping log levels and adding a lot of log statements to give me more details of what is going on.
3. Eliminate as much as possible variables that are not directly related to the issue at hand and that are not required to repro the issue, with the goal of getting to the bare minimum setup still being able to repro the issue
4. With a minimum repro setup, now is the time to form hipothesis, especulate what might be causing the issue, do a lot of resarch and, hopefully, find the root cause and a solution or workaround to the problem.


For this particular problem I first set out to understand what were all the variables involved in the original setup:

- Reviewed the GitHub Actions Workflow `.yml` file to understand the steps involved.
- The step that was causing the issue was running a PowerShell script, so I went to review that script and understand what it did.
- In doing that, I found out that the task within the Powershell script that was causing the issue was when it executed `npm run test` at a specific folder.
- I then reviewed the `package.json` on that folder to understand what it was doing, and that's where I found the `runtestps` script I mentioned above.

So at this point we have tons of moving parts: GitHub Actions Virtual Environment; the Previous GitHub Actions in the Workflow; Powershell; npm; `dotnet run`.
It was time to bump the logs and see what was going on. In doing so I found out that there were two `NPM_CONFIG` environment variables setup in the virtual environment before any step ran: `NPM_CONFIG_CACHE` and `NPM_CONFIG_PREFIX`.

After some time googling I found an [old issue on the npm repo](https://github.com/npm/npm/issues/14528) that gave me a north pointing to what could be causing the issue and also led me to a particular sentence in [node config docs regarding environment variables](https://docs.npmjs.com/misc/config#environment-variables), enphasis mine:

> Config values are case-insensitive, so NPM_CONFIG_FOO=bar will work the same [as npm_config_foo=bar]. However, **please note that inside npm-scripts npm will set its own environment variables and Node will prefer those lowercase versions over any uppercase ones that you might set**.

If you go read the issue, the gist of it is that within `npm-scripts` it will set all `npm_config_*` environment variables, all using lower case only, and they will supersede any upper case NPM_CONFIG_* environment variable that was previously defined. Although surprising, this is not a big problem per-se, or so I thought. The problem is that it does that, even when running on Windows, and it doesn't delete the upper case variable, it simply add the lower case one. This become a big problem on Windows because Environment Variables on Windows are **Case Insentitive** and there isn't even a way with Windows standard tooling to setup duplicate environment variables on Windows.


### Reducing the variables

With a hypothesis that I was fairly confident was the root cause for the issue, I went to eliminate all variables that were not directly related to the issue. Removed the dotnet project; the dotnet related GitHub Actions; reduced the build script to a bare minimum; and finally removed GitHub Actions altogether and got to a powershell one-liner that I could repro the issue locally on Windows:

In the directory with this `package.json`:

```json
{
    "scripts" : {
        "test" : "npm run env && @powershell Get-Item Env:"
    }
}
```

Run this Powershell one-liner

```powershell
$Env:NPM_CONFIG_CACHE='c:\temp';npm run test
```

The script will output the list of environment variables available in the context of npm-script and then would throw the same error when trying to list the environment variables using Powershell.

In the output of `npm run env` it lists this two environment variables:

```text
NPM_CONFIG_CACHE=C:\temp
npm_config_cache=C:\temp
```

Again, as we are on Windows, this is not supposed to be a possible state. At this point all bets are off.

## Fixing the issue, or at least having a workaround 

To recap the cause of the original issue:

- GitHub Actions Environment Windows-2019 with two environment variables defined by default: NPM_CONFIG_CACHE and NPM_CONFIG_PREFIX
- A GitHub Action Step that called `npm run` to run a _npm script_ that in turn executed `dotnet run` in a dotnet app that listed Environment Variables.
- npm script ignores upper case environment variables and create all lowercase ones using node's `process.env`, which somehow allow it to set "duplicate" environment variables on Windows differing only by upper/lower casing.
- This is not supposed to happen on Windows
- :boom:

I opened an [issue with npm/cli](https://github.com/npm/cli/issues/1527) to see if they can avoid this creating duplicate env variables on Windows to being with. In the mean time, to work around the issue, we can simply remove the UPPER CASE environment variables in our powershell build script before calling `npm run`:

```text
gci Env: | where Name -clike 'NPM_CONFIG_*' | remove-item

...

npm run test
```

And now I can go back to what I was supposed to be doing :)