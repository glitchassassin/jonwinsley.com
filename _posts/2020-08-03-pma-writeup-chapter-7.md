---
layout:     post
title:      "PMA Labs Writeup: Analyzing Malicious Windows Programs"
date:       2020-07-30 21:00:00
author:     Jon Winsley
comments:   true
summary:    My analysis of the disassembly labs in Chapter 7 of "Practical Malware Analysis".
categories: malware-analysis
---

[Practical Malware Analysis](https://practicalmalwareanalysis.com/) is still a handbook for aspiring malware analysts, and while I've dabbled in the subject before, I've decided to work through the book for a better hands-on grasp of malware reverse engineering. Needless to say, this writeup will contain spoilers.

# Chapter 7: Analyzing Malicious Windows Programs

## Lab 07-01

### Static Analysis

PEView reveals no immediate signs of packing or obfuscation. There are several imports of interest, including CreateService from advapi32.dll; CreateMutex and WriteFile from kernel32.dll; and InternetOpenUrl from wininet.dll. Interesting strings include:

```
MalService
malservice
HGL345
http://www.malwareanalysisbook.com
Internet Explorer 8.0
```

These seem likely to be service names, a mutex, and a URL and user-agent. Let's find out.

### Basic Dynamic Analysis

After running Lab07_01.exe, we immediately see two HTTP GET requests to the above domain, and we also see the exe installed as a service in the registry. Process Monitor reveals that it accesses our Temporary Internet Files, History, Cookies, etc., suggesting this malware may attempt to exfiltrate our data. There is no immediate sign of this in the logged HTTP requests, but it's possible the malware is looking for a command that inetsim isn't providing.

Further analysis is needed to confirm.

### Disassembly

The main function immediately calls StartServiceCtrlDispatcher, to connect to the service control manager, and then calls `sub_401040`.

This function uses the mutex `HGL345` to ensure only one process is started. Any additional instances of the process will exit immediately. Then it creates a SystemTime object for (based on the automatically labeled offsets) dayOfWeek 0 (Sunday), hour 0, seconds 0, and year 0x834 (2100 decimal); sets a timer; and waits. Once the timer goes off, the function spawns 20 threads in a loop and then sleeps effectively forever.

These threads jump to 0x401150, where they simply open a connection to http[:]//www[.]malwareanalysisbook.com in a loop indefinitely.

Based on this analysis, the malware appears to be a logic bomb, set to go off in 2100 and use up system resources. However, I was not able to set the Windows XP clock forward to 2100 - it only goes up to 2099! This, coupled with the fact that we *did* see requests for that URL, suggests we're misinterpreting the timer here.

Taking a closer look at the labels, we do indeed see that the initial offset is are different for each field, suggesting that the auto-generated structure labels don't quite line up:

![structure fields with different offsets](/assets/pma-7-1-1.png)

If we look at the offsets themselves, and calculate the structure manually from esp+4 (the starting address), we find that we are actually zeroing the entire structure with edx, and then assigning the value 0x0834 to the field +0x08 from the base of the structure. Based on the documentation, this is the Hour field.

If that's correct, this is an illegal value: wHour's maximum value is 23. Without some further debugging, I'm not positive what's going on here, so we'll circle back in our Wrap-up.

* Host-based indicators
  * Mutexes
    * HGL345
  * Services
    * Malservice
* Network-based indicators
  * Domains
    * http[:]//www[.]malwareanalysisbook.com

## Lab 07-02

### Static Analysis

This is a very compact file, though it doesn't appear to be packed. There are only a few imports of interest: OleInitialize, CoCreateInstance, and OleUninitialize from ole32.dll. There is one string of interest, suggesting a network component:

```
http://www.malwareanalysisbook.com/ad.html
```

### Basic Dynamic Analysis

After running the malware, Internet Explorer appears, pointed at the URL we saw in strings. No changes are immediately apparent in the Registry, and there does not appear to be any other significant network traffic. Nothing of immediate interest is highlighted by Process Monitor.

### Disassembly

