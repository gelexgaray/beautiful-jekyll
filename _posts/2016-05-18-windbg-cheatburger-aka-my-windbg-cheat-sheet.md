---
layout: post
title: "WinDBG CheatBurger (aka My WinDBG cheat sheet)"
date: 2016-05-26 
tags: ["WinDbg"]
comments: true
published: true
---

This is my personal cheat sheet. I recommend using it as a template to build your own... in this case one size does not fit all!

<!-- more -->
Take into account I personally use windbg to inspect memory dumps of dead processes, so, my cheat sheet is focused on this scenario.

## Initial setup
### Environment configuration

Configure symbol search path:	
```
	.sympath+ SRV*C:\symcache*http://msdl.microsoft.com/download/symbols
	.sympath+ C:\Users\GorkaE\OneDrive\WinDBG\Symdownload
```

Configure binary images and PDBs search path:
```
	.exepath c:\Path_To_Binaries
```
 

### Specific to minidumps

Recreate a full dump from a minidump:
```
	.dump /ma <path_to_full_dmp>
```

Recreate a full dump from a minidump, ignoring page read faults:
```
	.dump /mA <path_to_full_dmp>
```


### First inspection of the process 

Total running time of the process till the breakpoint was hit:
```
	.time
```
## .Net debugging extensions

Inspect WinDbg loaded extensions:
```
	.chain
```

Unload WinDbg extension
```
	.unload <extension>
```


### [SOS](<http://msdn.microsoft.com/en-us/library/bb190764(v=vs.110).aspx>)
	
*.Net Framework 4.0 and up*

Load sos.dll v4.0 64 bit<br/>
```
	.load C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SOS.dll
```
Load sos.dll v4.0 32 bit
```
	.load C:\Windows\Microsoft.NET\Framework\v4.0.30319\SOS.dll
```	
	
*.Net Framework older versions*
	
Load sos.dll v2.0 64 bit
```
	.load C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll
```
Load sos.dll v2.0 32 bit
```
	.load C:\Windows\Microsoft.NET\Framework\v2.0.50727\SOS.dll
```	

#### General information
Inspect information about loaded assemblies
```
	lm
	!lmi Assembly_Id
```
	
.Net runtime version
```
	lmv m mscor*
```	

#### Thread statistics
Threadpool general status:
```
	!threadpool
```

.Net special threads
```
	!threads -special
```

Accumulated running time of threads (good to find processing bottlenecks):
```
	!runaway
```

#### Thread inspection

Call stack of all current threads:
```
	~* e !CLRStack
```

Call stack of a thread, including method call parameters
(replace '1' with thread to inspect)
```
	!CLRStack -p
```	

> With sosex, 
> ```
> !mk -p
> ```
> shows this information better formatted.

Show call stack of managed threads waiting for unmanaged locks
```
	!EEStack -EE -short
```

Inspect value of an object
```
	!do 0xhexAddress
```
	
Get the de content of a "value type" (VT in the output of !do)
```
	dq 0xhexAddress+Offset L1 (64 bit)
	dd 0xhexAddress+Offset L2 (32 bit)
```
	
> With sosex, 
> ```
> !mdt 0xhexAddress
> ```
> shows this information better formatted.


Find objects in the call stack of a thread
```
	!dumpstackobjects 
```

Get Last Exception of all threads

```
	~* e !PrintException
```

Last unmanaged error of all threads
```
	~* e !gle
```

Find managed objects in the heap
```
	!dumpheap -stat
	!dumpheap -mt 0xhexAddress
```

> With sosex, there is way to filter objects taking into account their age:
> ```
>   !dumpgen 2 -stat
> ```
> .
> This will find all objects in GEN 2 Heap


> Tess Fernandez has an article about finding objects in the heap 
> [here](http://blogs.msdn.com/b/tess/archive/2005/11/25/496973.aspx?Redirected=true)

Find free areas in the heap (good to analyze heap fragmentation)
```
	!dumpheap -type Free
```

### [SOSEX](http://www.stevestechspot.com/SOSEXANewDebuggingExtensionForManagedCode.aspx)

Sosex help is quite self-explanatory and it can be considered a cheat sheet on itself
```
bhi       [filename]                                     BuildHeapIndex - Builds an index file for heap objects.
bpsc      (Deprecated.  Use !mbp instead)
chi                                                      ClearHeapIndex - Frees all resources used by the heap index and removes it from memory.
dlk       [-d]                                           Displays deadlocks between SyncBlocks and/or ReaderWriterLocks
dumpgen   <GenNum> [-free] [-stat] [-type <TYPE_NAME>]   Dumps the contents of the specified generation
                   [-nostrings]
finq      [GenNum] [-stat]                               Displays objects in the finalization queue
frq       [-stat]                                        Displays objects in the Freachable queue
gcgen     <ObjectAddr>                                   Displays the GC generation of the specified object
gch       [HandleType]...                                Lists all GCHandles, optionally filtered by specified handle types
help      [CommandName]                                  Display this screen or details about the specified command
lhi       [filename]                                     LoadHeapIndex - load the heap index into memory.
mbc       <SOSEX breakpoint ID | *>                      Clears the specified or all managed breakpoints
mbd       <SOSEX bmgureakpoint ID | *>                      Disables the specified or all managed breakpoints
mbe       <SOSEX breakpoint ID | *>                      Enables the specified or all managed breakpoints
mbl       [SOSEX breakpoint ID]                          Prints the specified or all managed breakpoints
mbm       <Type/MethodFilter> [ILOffset] [Options]       Sets a managed breakpoint on methods matching the specified filter
mbp       <SourceFile> <nLineNum> [ColNum] [Options]     Sets a managed breakpoint at the specified source code location
mdso      [Options]                                      Dumps object references on the stack and in CPU registers in the current context
mdt       [TypeName | VarName | MT] [ADDR] [Options]     Displays the fields of an object or type, optionally recursively
mdv       [nFrameNum]                                    Displays arguments and locals for a managed frame
mfrag     [-stat] [-mt:<MT>]                             Reports free blocks, the type of object following the free block, and fragmentation statistics
mframe    [nFrameNum]                                    Displays or sets the current managed frame for the !mdt and !mdv commands
mgu       // TODO: Document
mk        [FrameCount] [-l] [-p] [-a]                    Prints a stack trace of managed and unmanaged frames
mln       [expression]                                   Displays the type of managed data located at the specified address or the current instruction pointer
mlocks    [-d]                                           Lists all managed lock objects and CriticalSections and their owning threads
mroot     <ObjectAddr> [-all]                            Displays GC roots for the specified object
mt        (no parameters)                                Steps into the managed method at the current position
mu        [address] [-s] [-il] [-n]                      Displays a disassembly around the current instruction with interleaved source, IL and asm code
muf       [MD Address | Code Address] [-s] [-il] [-n]    Displays a disassembly with interleaved source, IL and asm code
mwaits    [-d]                                           Lists all waiting threads and, if known, the locks they are waiting on
mx        <Filter String>                                Displays managed type/field/method names matching the specified filter string
refs      <ObjectAddr> [-target|-source]                 Displays all references from and to the specified object
rwlock    [ObjectAddr | -d]                              Displays all RWLocks or, if provided a RWLock address, details of the specified lock
sosexhelp [CommandName]                                  Display this screen or details about the specified command
strings   [ModuleAddress] [Options]                      Search the managed heap or a module for strings matching the specified criteria

ListGcHandles - See gch

Use !help <command> or !sosexhelp <command> for more details about each command.
```

> From these, I would remark 
> ```
>	!dlk
> ```
> and 
> ```
>	!rwlock
> ```
> as functionalities not found in another libraries. You can survive without
> the rest of the operations, but generally, operations from sosex are better
> than their counterparts from sos or psscor


### [PSSCOR2](http://blog.coryfoy.com/2010/04/first-look-at-psscor2-the-new-windbg-debugging-extension-for-managed-code/) and [PSSCOR4](http://blogs.msdn.com/b/tom/archive/2011/04/28/now-available-psscor4-debugger-extension-for-net-4-0.aspx)

Psscor contains all the operations from sos, plus a lot of useful operations.
From the extended operation set, these are not found on the other libraries

ASP.NET cache statistics:
```
	!DumpASPNETCache -stat 
```

Dump entries in the ASP.NET cache
```
	!DumpASPNETCache -s 
```

A report of locks merging locks and waits. It gives you a very useful lock tree
```
	!SyncBlk
```

## Miscelaneous operations

Dump the output of WinDbg to a text file
```
	.logopen <path_to_file>
	.logclose
```

## Other cheat sheets from very smart people

1. https://theartofdev.com/windbg-cheat-sheet/
2. http://windbg.info/doc/1-common-cmds.html
