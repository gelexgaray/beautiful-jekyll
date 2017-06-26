---
layout: post
title: "Introduction to .net post-mortem debugging"
date: 2016-01-07
comments: true
tags: ["post-mortem", "debugdiag", ".net"]
---

There are lots of good practices to follow during software development. You can make, defensive programming, unit tests, load tests, manual testing, follow good design practices, follow agile methodologies… but whatever you do, the software will fail unexpectedly in the production environment.

<!-- More -->

## It’s almost impossible to replicate all the production environments

Your software will be installed with unknown third-party applications, with different network configurations and with different load patterns. It’s almost impossible to figure out all those variables while you are testing your software, so your application may crash in production. Many times, you will not be able to replicate this environment for a lab debugging session.

## Post mortem... What????

Here is when post-mortem debugging can help. Post mortem debugging is a set of techniques to diagnose a process using a memory dump. It’s like taking a photo of a process and take conclusions just but looking at it. You will have to forget all about debugging execution flow: you will see a frozen process that can’t be resumed, but all the information in the stack and heap will be there.

### What is this useful for?

This can be used to diagnose problems like:

- Stalled process (CPU usage is low, your process receives a workload, but it does nothing with it)
- High CPU usage during long periods of time
- Your process crashes without giving any error trace
- High memory usage
- Performance degradation over time
- … etc


### What is this NOT so useful for?

- Live process debugging: like knowing the value of a variable over the time.

## Memory dump types, when and how to take a snapshot
First of all, there are kernel mode dumps and user mode dumps. Kernel mode dumps are useful to debug driver related stuff, but we will concentrate on user mode dumps: those that are related to processes running on user space (those applications we usually program).

There are several user mode dump types in Windows, and further analysis options depend on which type you select.

- Full Dump: This dump type contains the whole memory address space of the dumped process. This is the most useful dump type and it should be used whenever possible. It has a little drawback: it’s the biggest one.
- Mini Dump: It’s a small dump that contains only data segments of the process. It’s small and it takes very little time to generate. As we only have data segments, we have to figure out the rest of the address space reconstructing code segments with the exact binaries that where binded to the process when we took the snapshot. These dump types are harder to debug, but sometimes we have no other possibilities. Windows Error Reporting tool generates this type of memory dumps.
- Custom Dump: It’s a Mini Dump with some configurable extra information. They are not common to see.

### 32bit vs 64bit Task Manager
On Windows 7 and later, memory dumps are generated using the Task Manager. Just right-click on the proper process, and select “Create Dump File”. It will generate a Full Dump on the temp folder.

There are two Task Managers on Windows 64 bit editions. You have to use the 32 bit one to generate 32 bit memory dumps. You can find 32 bit task manager on “C:\Windows\SysWOW64\taskmgr.exe”. If you use 64 bit task manager to dump 32 bit processes, it will successfully take a snapshot, but those snapshots will not load on most of the tools used to perform post-mortem analysis.

On older operating systems, you have to use third party tools like [Sysinternals Process Explorer](https://technet.microsoft.com/en-us/sysinternals/processexplorer.aspx)

### When to take a snapshot
- To debug CPU related issues, you have to take one snapshot when the process is stalled.
- To debug memory related issues, you have to take at least two snapshots: one when the process id fresh, and the other one when the process is degraded


### Automatic memory dumps
There are some tools that can take memory dumps automatically:

- Windows Error Reporting: It takes a minidump when a process dies with and unhandled exception
- Microsoft DebugDiag: This is a tool to perform automatic analysis on memory dumps. It has a service (DebugDiag Collection) that can be configured with rules to take automatic snapshots. Take a look at  [debugdiag.com](http://debugdiag.com). We will talk about this later.

## Debugging symbols
When you use an integrated IDE like Visual Studio, debugging seems a transparent task. You are directly looking at source code and it looks like if it was source code directly executing.

In the real world, machines execute binary code… and all you have in a dump is binary code. To interpret this, a mapping has to be done. Any source code line is compiled into a bunch of binary code instructions located in a RAM address. Taking a RAM address, with a proper map, you can know which source code instruction you are on. This “map” is located into the debugging symbols.

### It's all about symbols
With Visual Studio, you always have the correct symbols, and if you are attaching the debugger to a process with old or no symbols at all, you can’t debug it using source code view. With low level debuggers like WinDBG, you always can interpret any machine code with good or bad symbols. The better the symbols, the better will be the mapping, but you can debug with “approximate” symbols too. If you use bad symbols, you will know the “approximate” execution point in your source code.

To show where you are, low-level debuggers usually tell you a method name and an offset. They are telling you… “OK, I know your are ‘X’ RAM positions away from this point”. So take this into account:

- If method offset values are high, you are probably working with bad symbols.


### Looking for symbols

There are two ways to provide symbols:

- Using a symbol server
- Using locally stored *.pdb files

In both cases, you have to give a symbol search path. This can be a combination of symbol servers and local paths separated by ‘;’

```
SRV*C:\symcache*http://msdl.microsoft.com/download/symbols;C:\MySymbols
```

In this case:

- SRV*C:\symcache*http://msdl.microsoft.com/download/symbols: This will take symbols from Microsoft symbol server and store locally in “C:\symcache” folder
- C:\MySymbols: This is a local folder where I store PDB files


## Automatic analysis
Well, this post is arriving at its end, so let’s end it with something useful.

Previously, we talked about how to capture memory dumps with DebugDiag, we have talked about how and when to capture dumps, about symbol search paths… lets end all this with a simple ram dump analysis.

### DebugDiag Analysis

Microsoft DebugDiag installation creates two program shortcuts:

- DebugDiag Collection: This is used to configure automatic memory dump collection rules
- DebugDiag Analysis: This is a rule-based tool to analyze memory dumps.

When you start DebugDiag Analysis, you will see a screen like this:

![DebugDiag Analysis rule selection](/img/2015-12-15.1/DebugDiagAnalysis.jpg)

Now, it’s very easy to run a simple analysis. Simply tell DebugDiag which issue type would you like to analyze, select one or more memory dumps, and let it do the hard job.

Don’t forget to configure your symbol path! Microsoft symbol server is included by default, but you should configure any extra symbol path through the preferences dialog.

## It’s enough!

Well, I think it’s time to have a rest and take a time to do some diagnostics with DebugDiag. This has been a simple introduction: next time we well introduce a more advanced tool: windbg
