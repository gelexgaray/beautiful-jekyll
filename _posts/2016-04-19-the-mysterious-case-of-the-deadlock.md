---
layout: post
title: "The mysterious case of the Deadlock"
date: 2016-05-06
categories: ["WinDbg", ".net", "post-mortem", "real case" ]
comments: true
published: true
---
It was a promising day... A new shiny load test machine, the latest version of our software... We put all running together, and after some minutes the process got stuck. CPU usage was near 0%, we knew lots of requests were pending, but there was no productive work. Throughput 0 and an angry boss staring to your computer screen. What's going go?

<!-- More -->

It looks like some kind of deadlock, doesn't it? Well, there is no pain with a crash dump and WinDbg!

Let's start setting up the environment. We have to setup symbol search paths and helper extensions

```
0:000> .sympath+ SRV*C:\symcache*http://msdl.microsoft.com/download/symbols
Symbol search path is: SRV*C:\symcache*http://msdl.microsoft.com/download/symbols
Expanded Symbol search path is: srv*c:\symcache*http://msdl.microsoft.com/download/symbols
0:000> .sympath+ C:\Users\GorkaE\OneDrive\WinDBG\Symdownload
Symbol search path is: SRV*C:\symcache*http://msdl.microsoft.com/download/symbols;C:\Users\GorkaE\OneDrive\WinDBG\Symdownload
Expanded Symbol search path is: srv*c:\symcache*http://msdl.microsoft.com/download/symbols;c:\users\gorkae\onedrive\windbg\symdownload
0:000> .load C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SOS.dll
0:000> .load sosex
This dump has no SOSEX heap index.
The heap index makes searching for references and roots much faster.
To create a heap index, run !bhi
0:000> .load psscor2
```

Now, lets proof our perception: is the process really stuck?

```
0:000> !threadpool
Work Request in Queue: 24
--------------------------------------
Number of Timers: 3
--------------------------------------
CPU utilization 0%
--------------------------------------
Worker Thread: Total: 17 Running: 17 Idle: 0 MaxLimit: 2000 MinLimit: 8
Completion Port Thread:Total: 332 Free: 1 MaxFree: 16 CurrentLimit: 333 MaxLimit: 1000 MinLimit: 8
```

0% CPU and a lot of threads... hmm, this looks really strange. I love sosex's mwaits command: it will do the work and tell us which processes are locked

```

0:000> !mwaits
Examining SyncBlocks...
Scanning for ReaderWriterLock instances...

blah, blah blah...
more warnings and boring stuff...

ClrThread  DbgThread  OsThread    LockType    Lock              LockLevel
------------------------------------------------------------------------------
0x13a      277        0x78        SyncBlock   000000000e8a74d0                
0x49       165        0x90        RWLockSlim  000000008003fbb8  Reader        
0x16b      312        0x11c       SyncBlock   0000000007711f38                
0x16a      311        0x160       SyncBlock   000000000e8aba20                
0xfc       157        0x170       SyncBlock   00000000094bc050                

... more ...

0x1f       116        0x1140      SyncBlock   000000000786e190                
0x64       83         0x114c      RWLockSlim  000000008003fbb8  Reader        
0xe7       148        0x1160      RWLockSlim  000000008003fbb8  Reader        
0x47       30         0x1174      SyncBlock   0000000007871130                
0xb3       228        0x11c0      SyncBlock   000000000e8a74d0                

... and more ...

0xc1       78         0x131c      RWLockSlim  000000008003fbb8  Reader        
0x3a       111        0x1324      RWLockSlim  000000008003fbb8  Reader        
0x24       146        0x132c      RWLockSlim  000000008003fbb8  Reader        
0xfe       198        0x1330      SyncBlock   0000000009563e08                
0x16e      352        0x1338      SyncBlock   00000000077122a0                
0xc7       330        0x1364      SyncBlock   0000000007871130                
0x165      307        0x137c      SyncBlock   000000000e8aec48                
0x2f       347        0x13ac      SyncBlock   00000000097de268                
0x9        114        0x13b4      RWLockSlim  000000008003fbb8  Reader        

... well believe me, there were a lot of them ...

```

It looks like there are many threads waiting as readers for a ReaderWriterLockSlim, and some threads waiting for different locks around the code. Who has gained the access to the ReaderWriterLockSlim?

```
0:000> !mlocks

... blah, blah, blah ....

ClrThread  DbgThread  OsThread    LockType    Lock              LockLevel
------------------------------------------------------------------------------

... a lot of threads ...

0x1f       116        0x1140      RWLockSlim  000000008003fbb8  Writer        
0xc1       78         0x131c      SyncBlock   000000000786e190                
0xc1       78         0x131c      thinlock    00000001205768b8  (recursion:0)

... more, more, more ...

0x3a       111        0x1324      thinlock    00000000c0158c10  (recursion:0)
0x3a       111        0x1324      thinlock    00000000e00e76a0  (recursion:0)
0x24       146        0x132c      SyncBlock   0000000007629150                
0x24       146        0x132c      thinlock    00000000e0232cc8  (recursion:0)
0x9        114        0x13b4      SyncBlock   000000000ef9aff8                
0x9        114        0x13b4      thinlock    00000000a04c0848  (recursion:0)

... tons of threads owning locks ...

```

Look! Thread 116 is the owner of the ReaderWriterLockSlim, but it appears in both lists...
Thread 78 also appears in both lists! It really looks like a deadlock.

Sosex has a great command specific to detect deadlocks


```
0:000> !dlk
Examining SyncBlocks...
Scanning for ReaderWriterLock instances...
Scanning for holders of ReaderWriterLock locks...
Scanning for ReaderWriterLockSlim instances...
Scanning for holders of ReaderWriterLockSlim locks...
Examining CriticalSections...
Could not find symbol ntdll!RtlCriticalSectionList.
Scanning for threads waiting on SyncBlocks...
Scanning for threads waiting on ReaderWriterLock locks...
Scanning for threads waiting on ReaderWriterLocksSlim locks...
Scanning for threads waiting on CriticalSections...
*DEADLOCK DETECTED*
CLR thread 0x1f holds the Writer lock on ReaderWriterLockSlim 000000008003fbb8
...and is waiting for the lock on SyncBlock 000000000786e190 OBJ:00000000e0151330[System.Object]
CLR thread 0xc1 holds the lock on SyncBlock 000000000786e190 OBJ:00000000e0151330[System.Object]
...and is waiting for a Reader lock on ReaderWriterLockSlim 000000008003fbb8
CLR Thread 0x1f is waiting at Sisteplant.Captor.Server.MessagesBroker.MessagesBrokerServiceBase.SendMessage(Sisteplant.Captor.Server.MessagesBroker.Messages.CaptorMessage)(+0x6b IL,+0x1a6 Native)
CLR Thread 0xc1 is waiting at System.Threading.WaitHandle.WaitOne(Int64, Boolean)(+0x19 IL,+0x23 Native)


1 deadlock detected.

```
Wow!

So managed threads 116 and 78 are deadlocked. We should inspect both call stacks.

```
0:000> ~116 e !mk
... top secret call with a lot of top secret methods from my company ...

0:000> ~78 e !mk
... this call stack is also top secret ...
``` 

This way, we knew thread 116 was writing on a ReaderWriterLockSlim while waiting for a lock owned by thread 78.

Thread 78 was waiting for the thread 116 to complete the writing operation while owning the lock requested by thread 116.

Hmm... some sentence reordering and the problem was gone.

And here it comes the bonus command of today. RWLock structures are very prone to cause synchronization problems...
rwlock command from sosex can tell you the status of any rwlock (given its id). When invoked without arguments, rwlock gives a report of all RWLock structures found in the process.


```
0:000> !rwlock 000000008003fbb8
WriteLockOwnerThread:             0x1f
UpgradableReadLockOwnerThread:    None
ReaderCount:                      0
ReaderThreadIds:                  None
WaitingReaderCount:               125
WaitingReaderThreadIds:           0x2d,0x1b,0x17,0x44,0x57,0x66,0x7a,0x80,0x95,0xaa,0xb0,0xb8,0xc4,0xc6,0xdc,0x9f,0x92,0x90,0x86,0xc0,0xb6,0xa3,0x70,0x45,0x39,0x84,0x9c,0x72,0x67,0x6a,0x65,0x60,0xdf,0xe0,0xe2,0x7e,0xa0,0xcf,0x4d,0x61,0xcd,0xc1,0xab,0x8f,0x64,0x4f,0x4a,0x48,0x1a,0x23,0x6f,0xd9,0xaf,0x62,0xd3,0xd0,0x2e,0x22,0x12,0xbe,0x52,0xae,0x8d,0x3a,0x1c,0x9,0x3d,0x5c,0x78,0x88,0xad,0xdb,0xd7,0xce,0xb4,0xa5,0x9d,0x85,0x74,0x6c,0x43,0xb,0x16,0x20,0x34,0xc,0x24,0x32,0xe7,0xe8,0xf2,0xf3,0xf4,0xf7,0xf9,0xfa,0xfd,0x40,0x7,0xf5,0x5e,0x3b,0x3e,0x49,0x5a,0xa8,0x14,0x9b,0x69,0x19,0x2c,0x13,0xd2,0x99,0x56,0x4c,0x11,0x54,0x98,0xb9,0xbb,0xc8,0x53,0xe3,0x42
WaitingWriterCount:               125
WaitingWriterThreadIds:           None
WaitingUpgradableReaderCount:     0
WaitingUpgradableReaderThreadIds: None
WaitingWriterUpgradeCount:        0
WaitingWriterUpgradeThreadIds:    None
```

As you can see, the ReaderWriterLockSlim owned by thread 116 was a very big bottleneck for the application, so, we finally ended up removing this ReaderWriterLockSlim...
But this is another good history for another post.

Stay tuned! Next time I will show you my cheat sheet for WinDbg.
