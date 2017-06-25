---
layout: post
title: "The sad story of the orphaned ReaderWriterLockSlim"
date: 2016-07-19 14:16:31 +0200
comments: true
categories: ["WinDbg", "real case", "post-mortem", ".net internals"]
---
Another day in the operations arena. Everything was working correctly, and performance counters had great values. CPU usage was near 0%... system idle. Suddenly a phone call and an angry voice from the other side. What??? The system seems down???

<!-- More -->

## Introducing the case

The most interesting fact here was that there was no activity in the analyzed process. Normally, this should indicate a lock issue, but this behavior was lasting a lot of time. Well... perhaps a deadlock 

```
0:000> .time
Debug session time: Mon Dec 21 10:25:58.000 2015 (UTC + 1:00)
System Uptime: 16 days 23:18:00.314
Process Uptime: 0 days 0:25:00.000
  Kernel time: 0 days 0:03:00.000
  User time: 0 days 0:14:52.000
0:000> .load C:\Windows\Microsoft.NET\Framework64\v2.0.50727\SOS.dll
0:000> !threadpool
CPU utilization 0%
Worker Thread: Total: 8 Running: 1 Idle: 7 MaxLimit: 2000 MinLimit: 8
Work Request in Queue: 0
--------------------------------------
Number of Timers: 3
--------------------------------------
Completion Port Thread:Total: 159 Free: 2 MaxFree: 16 CurrentLimit: 160 MaxLimit: 1000 MinLimit: 8
```
> The CPU was on vacation, as you can see

The first thing to do when you suspect threads are locked, is to inspect all the "lock tree".  

```
0:000> !SyncBlk
Index         SyncBlock MonitorHeld Recursion Owning Thread Info          SyncBlock Owner
 7748 000000000913e308            1         1 0000000009464cd0  1f74  79   000000014033da70 System.Object
		Waiting threads:
 8591 0000000008cd3a40            1         1 000000000539bbc0  1628  25   0000000100544688 System.Object
		Waiting threads:
 8655 0000000008c7ea10            1         1 00000000092c0c40   df4 127   00000000e039e980 System.Object
		Waiting threads:
 8815 0000000008be1e00            1         1 000000000dfff8f0  1464  88   0000000100497100 System.Object
		Waiting threads:
 9313 0000000008fe1f48            1         1 000000000e419e30  1fb8  59   00000000e044fbf0 System.Object
		Waiting threads:
 9320 000000000914ef98            1         1 000000000924db20  1760  35   00000000a0272d78 System.Object
		Waiting threads:
 9446 0000000009132520            1         1 000000000db84e60  211c 160   00000001201b5040 System.Object
		Waiting threads:
 9943 0000000008e22e10            1         1 000000000db84890  1d94 165   000000010045bb10 System.Object
		Waiting threads:
 9944 0000000008e11e20            1         1 000000000db882b0  23b0 158   000000010045ba68 System.Object
		Waiting threads:
 9948 0000000009136758            1         1 00000000052153d0  1758 126   0000000080270378 System.Object
		Waiting threads:
 9955 0000000009405dc8            1         1 000000000da79350  1458 148   00000001402ddc38 System.Object
		Waiting threads:
10541 0000000008e11a78            1         1 000000000da79920   c14 149   00000001201b4f98 System.Object
		Waiting threads:
10551 0000000008a8da60            1         1 000000000da7aa90  1fa0 151   00000000e039ef40 System.Object
		Waiting threads:
10593 0000000008e29580            1         1 000000000e178c40  1aa0 143   000000010045aa88 System.Object
		Waiting threads:
10604 0000000008fd3e20            1         1 000000000e1797e0  195c 119   00000000e039e590 System.Object
		Waiting threads:
10608 0000000008c80490            1         1 000000000da75360   420 146   00000001601e0ef8 System.Object

... Lots of similar entries ...

		Waiting threads:
-----------------------------
Total           18154
CCW             4
RCW             433
ComClassFactory 1
Free            16028
```

Look, there were a lot of threads locked, but there wasn't a clear locking thread. 
What kind of locks were there? 

```
0:000> !mlocks
Examining SyncBlocks...
Scanning for ReaderWriterLock instances...
Scanning for holders of ReaderWriterLock locks...
Scanning for ReaderWriterLockSlim instances...
Scanning for holders of ReaderWriterLockSlim locks...
Examining CriticalSections...
Could not find symbol ntdll!RtlCriticalSectionList.

ClrThread  DbgThread  OsThread    LockType    Lock              LockLevel
------------------------------------------------------------------------------
0xaa       -1         0xffffffff  RWLockSlim  0000000160125d70  Writer        
0xa6       150        0x11c       thinlock    0000000080270878  (recursion:0)
0xa6       150        0x11c       thinlock    0000000140402dd0  (recursion:0)
0x112      53         0x158       SyncBlock   0000000008e22750                
0x112      53         0x158       thinlock    0000000140343ab8  (recursion:0)
0x13e      71         0x340       SyncBlock   0000000008a82e30                
0x13e      71         0x340       thinlock    00000001000b2550  (recursion:0)
0xca       146        0x420       SyncBlock   0000000008c80490                
0xca       146        0x420       thinlock    0000000080409640  (recursion:0)
0x131      122        0x4b4       SyncBlock   0000000008e20580                
0x131      122        0x4b4       thinlock    00000000c0238a88  (recursion:0)
0x104      52         0x5f0       thinlock    000000016029d250  (recursion:0)
0x104      52         0x5f0       thinlock    000000016029e858  (recursion:0)

... and many, many, many more ...

```

> The command 
> ```
> mlocks
> ```
> from sosex gives a perspective on which lock types are holding threads 

We see locks of three types:

- SyncBlock: These are regular "monitor" objects. Usually, a "lock" on a variable gives you a SyncBlock.

- [Thinlock](http://www.c2.com/cgi/wiki?ThinLocks): It's an optimization over regular monitors. To simplify, let's say they are lighter primitives than SyncBlocks. When you use a lock and only a single thread owns this lock, a thinlock is created. Later, when more threads request the same lock, this thinlock promotes to a SyncBlock. In our dump, there were a lot of threads gaining some kind of lock, but not blocking any other thread. 

- RWLockSlim: It is a special object to synchronize access of "readers" and "writer" threads. The theory behind this artifact is quite easy: reader threads only block writer threads, and writer threads block any other thread. 

But... look!  What the hell is a -1 thread???

It is holding a writer lock on a RWLockSlim... so, it is a writer thread (remember it will be blocking any other reader or writer thread requesting the same lock)

### A positive look to a negative thread

Let's look what's happening on this -1 mysterious thread

```
0:000> ~-1 e !mk
        ^ Syntax error in '~-1 e !mk'
```

OK, it is really weird, isn't it?

Well.. perhaps it is not so weird if it look at the RWLockSlim hold by this strange thread

> The command
> ```
> rwlock
> ```
> from sosex tells you which threads are waiting to read or write on a rwlock


```
0:000> !rwlock 0000000160125d70
*** WARNING: Unable to verify checksum for System.ni.dll
WriteLockOwnerThread:             0xaa (DEAD)
UpgradableReadLockOwnerThread:    None
ReaderCount:                      0
ReaderThreadIds:                  None
WaitingReaderCount:               151
WaitingReaderThreadIds:           0x2e,0x3e,0x2b,0x1f,0xc,0x9,0x44,0x4b,0x4e,0x2a,0x6d,0x99,0x57,0x62,0x3b,0xa7,0xb3,0xb5,0xc0,0xce,0x24,0x41,0xe2,0xf0,0xf2,0x101,0x104,0x112,0x120,0x121,0xf3,0x94,0xd6,0x98,0x97,0x36,0x126,0x128,0x12b,0x136,0x137,0x139,0x13c,0x13e,0x142,0x145,0x14b,0x14c,0x144,0x12e,0xef,0xe6,0xf6,0x88,0x13f,0x13b,0x135,0x12a,0x129,0x13,0x4a,0x5e,0x9d,0xbf,0xc6,0xcf,0x5a,0x70,0x125,0x124,0x123,0x122,0x11e,0x11a,0x116,0x115,0x10c,0x10b,0x106,0xed,0x91,0x5c,0xe,0x119,0x127,0x12d,0x13a,0x52,0xd5,0xf8,0x12c,0x131,0x133,0x11d,0x13d,0x146,0x143,0x130,0x59,0xa1,0xbc,0xcb,0xfc,0x6f,0xc9,0xba,0x29,0x3,0x1a,0x3d,0x4,0x74,0x76,0x80,0xa0,0xca,0xc7,0xad,0xab,0xa6,0xa2,0x84,0x81,0x7f,0x7d,0x38,0x40,0x39,0x58,0x64,0x60,0x4c,0x7,0x21,0x32,0x117,0x114,0x111,0xfa,0x10f,0x10e,0x10d,0x109,0x108,0x107,0x105,0x103,0x102,0x100,0xff,0xfe
WaitingWriterCount:               151
WaitingWriterThreadIds:           None
WaitingUpgradableReaderCount:     0
WaitingUpgradableReaderThreadIds: None
WaitingWriterUpgradeCount:        0
WaitingWriterUpgradeThreadIds:    None
*This lock's writer lock is orphaned.
```

Here you have the answer... thread 0xAA died while it was holding the ReaderWriterLock with a write lock.

### But is it possible?

Of course it is possible, and it happens as you can see... so, remember:

- Lock sentences are easier to use in C#, because locks are automatically released in case of an exception (it is a pattern similar to the using statement)
- If you use more complex primitives like the RWLockSlim, whenever you request a lock, be sure it is always released on a finally block.

My mind is focused on summer vacation, and I don't know when and on what am I going to talk on next post. Perhaps something on LOH or some basic article with a simple guide for dummies to diagnose application crashes with debugdiag... it would be great if every developer on the earth use debugdiag before asking for help to debug a crash!

