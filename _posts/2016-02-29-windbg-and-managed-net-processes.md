---
layout: post
title: "WinDbg and managed .NET processes"
date: 2016-03-21
comments: true
tags: ["WinDbg", ".net", "post-mortem" ]
---
Ok, so you are a managed code developer and somebody told you WinDbg is an unmanaged code debugger... Well, in fact, it is, but there are plenty of extensions that make possible debugging managed processes. 
<!-- More -->

The command used to load libraries is ".load". Libraries are searched in the WinDbg folder and in the computer PATH environment folders. To provide a library in another folder, its full path must be provided.

<blockquote>
Tip: If you type ".chain", you will see the full extension search path in the top of the output
</blockquote>

## SOS
No, I'm not asking for help. SOS.dll is the standard Microsoft extension library to help WinDbg with managed code. 

This library comes with the .net framework redistributable, and it is automatically located in the .net framework installation path. It could be something similar to:

- C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll (2.0 to 3.5 Frameworks, 64 bit)
- C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SOS.dll (4.x Frameworks, 64 bit)
- C:\Windows\Microsoft.NET\Framework\v2.0.50727\SOS.dll (2.0 to 3.5 Frameworks, 32 bit)
- C:\Windows\Microsoft.NET\Framework\v4.0.30319\SOS.dll (4.x Frameworks, 32 bit)
- etc 

Normally, ".load sos" should load the correct version, but you could also force loading a specific version using the full path to the library

```
0:000> .load sos
0:000> !sos.help
-------------------------------------------------------------------------------
SOS is a debugger extension DLL designed to aid in the debugging of managed
programs. Functions are listed by category, then roughly in order of
importance. Shortcut names for popular functions are listed in parenthesis.
Type "!help <functionname>" for detailed info on that function. 

Object Inspection                  Examining code and stacks
-----------------------------      -----------------------------
DumpObj (do)                       Threads
DumpArray (da)                     CLRStack
DumpStackObjects (dso)             IP2MD
DumpHeap                           U
DumpVC                             DumpStack
GCRoot                             EEStack
ObjSize                            GCInfo
FinalizeQueue                      EHInfo
PrintException (pe)                COMState
TraverseHeap                       BPMD 

Examining CLR data structures      Diagnostic Utilities
-----------------------------      -----------------------------
DumpDomain                         VerifyHeap
EEHeap                             DumpLog
Name2EE                            FindAppDomain
SyncBlk                            SaveModule
DumpMT                             GCHandles
DumpClass                          GCHandleLeaks
DumpMD                             VMMap
Token2EE                           VMStat
EEVersion                          ProcInfo 
DumpModule                         StopOnException (soe)
ThreadPool                         MinidumpMode 
DumpAssembly                       
DumpMethodSig                      Other
DumpRuntimeTypes                   -----------------------------
DumpSig                            FAQ
RCWCleanupList
DumpIL
```

<blockquote>
SOS is the less powerful .net diagnostics extension, but it requires no deployment, and it's always available on any computer with the .net Framework installed
</blockquote>

### Common pitfalls with SOS
To debug with SOS extensions, the .net framework version in the debugging machine must be the same as the target machine. This could not look like a problem to debug live processes, but think on memory dumps: it's very common to have dumps from computers with a different patchlevel than yours.

In these cases, you will see an error like this:
```
0:000> .load sos
Failed to load data access DLL, 0x80004005
```

[Here](https://blogs.msdn.microsoft.com/dougste/2009/02/18/failed-to-load-data-access-dll-0x80004005-or-what-is-mscordacwks-dll/ ) you can read a full explanation on what's happening. I will condense the most important information:

- sos.dll depends on mscordacwks.dll
- Normally, if you have properly configured your symbol search path, the correct mscordacwks.dll will be loaded automatically
- Sometimes, Microsoft guys don't publish all the symbols of all of its .net framework versions. They should... but they don't
- In these cases, you must manually get the correct mscordacwks.dll from the target computer and drop it to a local folder. The downloaded dll should be named this way: "mscordacwks_AAA_AAA_u.v.wwww.xxxx.dll"
- u.v.wwww.xxxx is the version number of the framework
- AAA_AAA is the architecture of the library (AMD64_AMD64 or x86_x86) 

I usually collect all the mscordacwks dlls I get, and put them on a directory I add to WinDBG symbol search path

```
.sympath+ C:\Symdownload
```

If you work at a company like mine, probably it will very hard to get the mscordawcks.dll directly from your target computer. Here is where the '[Mscordacwks/SOS debugging archive](http://sos.debugging.wellisolutions.de/)' comes your rescue. Another way to find missing mscordwcks dlls is extracting them from Microsoft Update packages as explained [here](https://chentiangemalc.wordpress.com/2014/04/16/obtaining-correct-mscordacwks-dll-for-net-WinDbging/).

## PSSCOR2 and PSSCOR4
[PSSCOR2](http://blogs.msdn.com/b/tom/archive/2010/03/29/new-debugger-extension-for-net-psscor2-released.aspx) and [PSSCOR4](http://blogs.msdn.com/b/tom/archive/2011/04/28/now-available-psscor4-debugger-extension-for-net-4-0.aspx) are modern and higher level extension libraries from Microsoft. They have a lot of utilities that are not present in SOS extension and that can save a lot of work. In fact, this extension has the total functionality of SOS plus a lot of additional stuff.

The only issue with this extension is that it has to be downloaded from Microsoft and put on the WinDbg extension search path. The easiest way to deploy this extension is to copy it on the WinDbg folder.

<blockquote>
The unique difference between PSSCOR2 and PSSCOR4 is the target .Net framework. You should use PSSCOR2 for applications running on frameworks 2.x and 3.x. PSSCOR4 should be used on higher versions of the .Net framework. 
</blockquote>


```
0:000> .load psscor2
0:000> .chain
Extension DLL chain:
    psscor2: image 2.0.0.1, API 1.0.0, built Wed Mar 24 20:25:01 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\psscor2.dll]
0:000> !psscor2.help
-------------------------------------------------------------------------------
PSSCOR is a debugger extension DLL designed to aid in the debugging of managed
programs. Functions are listed by category, then roughly in order of
importance. Shortcut names for popular functions are listed in parentheses.
Type "!help <functionname>" for detailed info on that function. 

Object Inspection                  Examining code and stacks
-----------------------------      -----------------------------
DumpObj (do)                       Threads
DumpArray (da)                     CLRStack
DumpStackObjects (dso)             IP2MD
DumpAllExceptions (dae)            BPMD
DumpHeap                           U
DumpVC                             DumpStack
GCRoot                             EEStack
ObjSize                            GCInfo
FinalizeQueue                      EHInfo
PrintException (pe)                COMState
TraverseHeap
DumpField (df)
DumpDynamicAssemblies (dda)
GCRef
DumpColumnNames (dcn)
DumpRequestQueues
DumpUMService

Examining CLR data structures      Diagnostic Utilities
-----------------------------      -----------------------------
DumpDomain                         VerifyHeap
EEHeap                             DumpLog
Name2EE                            FindAppDomain
SyncBlk                            SaveModule
DumpThreadConfig (dtc)             SaveAllModules (sam)
DumpMT                             GCHandles
DumpClass                          GCHandleLeaks
DumpMD                             VMMap
Token2EE                           VMStat
EEVersion                          ProcInfo 
DumpModule                         StopOnException (soe)
ThreadPool                         MinidumpMode 
DumpHttpRuntime                    FindDebugTrue
DumpIL                             FindDebugModules
PrintDateTime                      Analysis
DumpDataTables                     CLRUsage
DumpAssembly                       CheckCurrentException (cce)
RCWCleanupList                     CurrentExceptionName (cen)
PrintIPAddress                     VerifyObj
DumpHttpContext                    HeapStat
ASPXPages                          GCWhere
DumpASPNETCache (dac)              ListNearObj (lno)
DumpSig
DumpMethodSig                      Other
DumpRuntimeTypes                   -----------------------------
ConvertVTDateToDate (cvtdd)        FAQ
ConvertTicksToDate (ctd)
DumpRequestTable
DumpHistoryTable
DumpBuckets
GetWorkItems
DumpXmlDocument (dxd)
DumpCollection (dc)

Examining the GC history
-----------------------------
HistInit
HistStats
HistRoot
HistObj
HistObjFind
HistClear
```

## SOSEX
[SOSEX](http://www.stevestechspot.com/SOSEXANewDebuggingExtensionForManagedCode.aspx) is an extension library from Steve's Techspot. It is not from Microsoft but it is really nice. Some functionality is overlapped with PSSCOR and SOS, but it has some nice utilities not present in neither of the other libraries. It's a must have if you can install extra libraries.

```
0:000> .load sosex
This dump has no SOSEX heap index.
The heap index makes searching for references and roots much faster.
To create a heap index, run !bhi
0:000> !sosex.help
SOSEX - Copyright 2007-2012 by Steve Johnson - http://www.stevestechspot.com/
To report bugs or offer feedback about SOSEX, please email sjjohnson@pobox.com
Quick Ref:
--------------------------------------------------
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
mbd       <SOSEX breakpoint ID | *>                      Disables the specified or all managed breakpoints
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

<blockquote>
SOSEX is really good to find objects on the heap. It can build an index for fast search of objects and it allows "jumping" between object references using hyperlinks. It also has a really nice interlock detection algorithm.
</blockquote>

# Conclussions
I've presented you three extensions to inspect managed processes, but you don't need to choose only one. I personally use the three of them. In the following posts we will use them to diagnose common .Net application problems.

See you in the next posts!
