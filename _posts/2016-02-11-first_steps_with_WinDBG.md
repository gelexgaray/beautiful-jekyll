---
layout: post
title: "First steps with WinDbg"
date: 2016-02-23
comments: true
categories: ["WinDbg"]
---
WinDbg is the Microsoft's "de-facto" standard for kernel-mode debugging. It sounds like a hacking tool, or something restricted to gurus... but it can be really useful for "mainstream developers" like you and me. It can be used to debug user-mode code, it has capabilities to debug .NET processes, and best of all: it's free (as in "free beer").
<!-- More -->

## Getting and installing WinDbg
WinDbg is part of Windows SDK (Software Development Kit) and WDK (Windows Driver Kit). If you don't want to install all the Windows SDK components, you can choose to install only "Windows Debugging Tools" during the installation process.

Take into account that there are two versions of WinDbg: the 32 bit one and the 64 bit one. As you probably guessed, to debug 32 bit processes you must use 32 bit version, and to debug 64 bit processes you must use 64 bit version.

So, you will probably want to install both of them. In this case, two program group folders are created.

## First impression
When you launch the proper version to debug your process, you will see a "nice" user interface. Don't get confused... WinDbg is mainly a command driven debugger. Very few of the options are available through the menus. Don't worry, i don't know all the WinDgb commands, and probably nobody knows. I usually use a "cheat sheet" with the commands I use and with some procedures to diagnose common problems. My advise is to build your own custom cheat sheet.

### The empty screen syndrome
When you launch WinDbg, you will not see any window to type in commands. You will have to start "finding your way" on an empty interface. To type commands you will usually do one of this two things:

- Usually you will want to debug a memory dump... so, first step is going to "file" menu and opening a crash dump.
- You can also attach to an existing process (File/Attach to a process)

### WinDbg command classification

There are three type of commands in WinDbg

- **Dot started commands**: These commands are directed to WinDbg itself. Some of them are available through WinDbg menus. They are used to configure the environment and the debugging session (to change the debugging symbols path, to clear the screen, to attach to a process... etc)
- **Regular commands**: These are the "normal" commands sent to de debugging engine (to put a breakpoint, to stop, to continue, to inspect a memory address content... etc). A process has to be attached or a memory dump loaded.
- **Exclamation mark started commands**: This commands are handled through WinDbg extensions. We will talk later on this, because they are mandatory to debug .NET processes.

There are also another special characters, like "~". These act like **operators**. In this case ~ means "select thread number X". With this operators and commands, you can build expressions like

```
~1 e k
```

*"For the first thread, execute command k, that shows the unmanaged call stack"*

### A simple example

Now let's try it... Launch notepad, launch "WinDbg" and select "File\Attach to a process". Select "notepad.exe" from the process list (it will probably be located close to the last one). Press "OK".

#### Invasive vs non invasive mode

There are two options when you attach to a process: attach in invasive or non invasive mode. If you use invasive mode, a complete debugging experience is available.

Non invasive allows attaching multiple debuggers to a single process. It's more like "instrumenting" a process: you can see it and inspect its memory but you cannot control the flow (it is more similar to analyzing a memory dump).

#### Let's continue with the example...

Let's try the default "invasive mode" debugging experience.

What happens? Try to switch to notepad application... it doesn't respond, does it? Go back to WinDbg and type "k". You will see current call stack of notepad

```
0:001> k
Child-SP          RetAddr           Call Site
00000000`0227fa78 00000000`77b040b8 ntdll!DbgBreakPoint
00000000`0227fa80 00000000`77805a4d ntdll!DbgUiRemoteBreakin+0x38
00000000`0227fab0 00000000`77a3b831 kernel32!BaseThreadInitThunk+0xd
00000000`0227fae0 00000000`00000000 ntdll!RtlUserThreadStart+0x1d
```

Ok, so we have correctly attached and the process has been interrupted

Now type "g" and let the notepad process continue. The notepad interface will now be responsive again.

Press "Ctrl+Break" to break into debug mode. If you type "k" again you will see the current call stack

#### Symbols, symbols, symbols
As we said in previous posts, configuring symbol path is the most important step to have a good debugging experience.

Type the following commands

```
.sympath+ SRV*C:\symcache*http://msdl.microsoft.com/download/symbols
.reload
```

This will point to Microsoft public symbol server, set a local cache for downloaded symbol files in "c:\symcache" and reload all necessary symbols given the new symbol path.

This operation can also be executed from the "File\Symbol File Path" menu option.

If you type "k" again, you will probably not see too much difference with the previous call stack, but with more complex scenarios it is vital to have a good symbol search path.

#### Freedom for notepad!
To finish the debugging session, two options are available:

- Type ".detach".
- Use "Debug\Detach debuggee" menu option.

#### Help commands
Wow, commands again?? this seems too hard to memorize.

Yes, it is... but you have several help commands to remember what you can do with this tool

- ?: Shows debugger regular commands (those not started with "." or "!"). Here you will see the "k" command we used before.
- .help: Shows WinDbg commands. Here you will see the ".detach" command we used before.

## Extensions
Now type ".chain" into a WinDbg command window

```
0:001> .chain
Extension DLL chain:
    C:\Program Files\Debugging Tools for Windows (x64)\sosex.dll: image 4.5.0.0, API 1.0.0, built Wed Oct 03 16:57:55 2012
        [path: C:\Program Files\Debugging Tools for Windows (x64)\sosex.dll]
    C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll: image 2.0.50727.5485, API 1.0.0, built Wed Jun 18 07:13:05 2014
        [path: C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll]
    dbghelp: image 6.12.0002.633, API 6.1.6, built Mon Feb 01 21:15:44 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\dbghelp.dll]
    ext: image 6.12.0002.633, API 1.0.0, built Mon Feb 01 21:15:46 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\winext\ext.dll]
    exts: image 6.12.0002.633, API 1.0.0, built Mon Feb 01 21:15:38 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\WINXP\exts.dll]
    uext: image 6.12.0002.633, API 1.0.0, built Mon Feb 01 21:15:36 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\winext\uext.dll]
    ntsdexts: image 6.1.7650.0, API 1.0.0, built Mon Feb 01 21:15:18 2010
        [path: C:\Program Files\Debugging Tools for Windows (x64)\WINXP\ntsdexts.dll]
```

Your extension chain may look different.

#### What is this for??
Well, as we said before, some of the WinDbg commands (those started with "!") are called *extension commands*. This commands are not directly handled by WinDbg: they are handled by extension libraries (those present in the ".chain" command output).

Some commands may be defined in various extension libraries: the first extension in the chain that defines a command is the extension used by default to handle it.

#### !help
No, I have not mistyped the exclamation mark. !help is the command usually used to request help about commands implemented in an extension library. Logically, if you type !help and all the extensions implement a help command, you will see the output of the first library in the chain. So, with my chain, if I type !help, I will get help about sosex library implemented commands.

To force the execution of a command on a certain library, you have to prepend library's name

```
0:001> !ext.help
diskspace [:]              - Displays free disk space for specified volume
address [address]          - Displays the address space layout
        [-UsageType]       - Displays the address space regions of the given type
analyze [-v]               - Analyzes current exception or bugcheck
cpuid [processor]          - Displays CPU version info for all CPUs
elog_str                   - Logs simple message to host event log
cppexr                     - Displays a C++ EXCEPTION_RECORD
error [errorcode]          - Displays Win32 & NTSTATUS error string
exchain                    - Displays exception chain for current thread
for_each_frame <cmd>       - Executes command for each frame in current
                             thread
for_each_local <cmd> $$<n> - Executes command for each local variable in
                             current frame, substituting fixed-name alias
                             $u<n> for each occurrence of $$<n>
gle [-all]                 - Displays last error & status for current thread
imggp                      - Displays GP directory entry for 64-bit image
imgreloc                   - Relocates modules for an image
list [-? | parameters]     - Displays lists
obja                       - Displays OBJECT_ATTRIBUTES[32|64]
owner [symbol!module]      - Detects owner for current exception or
                             bugcheck from triage.ini
rtlavl                     - Displays RTL_AVL_TABLE
std_map <address>          - Displays a std::map<>
str <address>              - Displays ANSI_STRING or OEM_STRING
ustr                       - Displays UNICODE_STRING
Type ".hh [command]" for more detailed help
```

## Bonus command
Wow... we have learned a lot today. To leave this post happy, I will tell you my bonus command of today.

```
!analyze -v
```

This extension gives you an analysis of what is the current exception of the process. Test it on any attached process and it will give you something similar to

```
EXCEPTION_RECORD:  ffffffffffffffff -- (.exr 0xffffffffffffffff)
ExceptionAddress: 000000007724cc90 (ntdll!DbgBreakPoint)
   ExceptionCode: 80000003 (Break instruction exception)
  ExceptionFlags: 00000000
NumberParameters: 1
   Parameter[0]: 0000000000000000
```

Nothing useful: it says the process is on a breakpoint.

But think about it... if you use this command with a memory dump taken with debugdiag or left by the operating system on a process crash, you will have a first clue of what made this process crash. Sometimes when you have no traces or eventlog information about a process crash, this command can be a life saver.

Ok, now go and practice.
Next time I will tell you about some extensions useful to debug .net processes 
