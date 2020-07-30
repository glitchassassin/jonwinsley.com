---
layout:     post
title:      "PMA Labs Writeup: IDA Pro"
date:       2020-07-29 15:00:00
author:     Jon Winsley
comments:   true
summary:    My analysis of the disassembly labs in Chapter 5 of "Practical Malware Analysis".
categories: malware-analysis
---

[Practical Malware Analysis](https://practicalmalwareanalysis.com/) is still a handbook for aspiring malware analysts, and while I've dabbled in the subject before, I've decided to work through the book for a better hands-on grasp of malware reverse engineering. Needless to say, this writeup will contain spoilers.

# Chapter 5: Basic Dynamic Analysis

For this chapter, I had to track down an older version of IDA Freeware that would still run on Windows XP. Luckily the fine folks at ScummVM got permission from HexRays to host [an older version, 5.0](https://www.scummvm.org/news/20180331/), which happens to coincide with the version used in the book. It's IDA Freeware, not IDA Pro, but we'll just have to make do.

## Lab 05-01

In this writeup, I'll follow the script of the questions a little more closely, to make sure I don't miss anything significant.

**What is the address of `DllMain`?**

We begin with `DllMain`. After loading the dll into IDA, `DllMain` is right in front of my face, and after enabling line prefixes in the General Options I can see it starts at address `1000D02E`.

**Use the Imports window to browse to `gethostbyname`. Where is the import located?**

The import appears in the .idata section, at address `100163CC`.

**How many functions call `gethostbyname`?**

IDA provides a graph of cross-references that call `gethostbyname`. There are five separate subroutines identified.

**Focusing on the call to `gethostbyname` located at 0x10001757, can you figure out which DNS request will be made?**

Immediately before the call, a memory address is moved into `eax` and then incremented by 0x0D (decimal 13). Looking up the memory address, we find the string `[This is RD0]pics.practicalmalwareanalysis.com`. Because the address was incremented by 13 bytes, we remove the first 13 characters and are left with `pics.practicalmalwareanalysis.com`.

**How many local variables has IDA Pro recognized for the subroutine at 0x10001656?**

This is the same subroutine we were just in. Moving the graph back to the top, I count twenty variables, recognizable by a negative offset. Some are prefixed, like `var_675`, and some have names apparently inferred, like `hModule`.

**How many parameters has IDA Pro recognized for the subroutine at 0x10001656?**

There is one parameter, with a positive offset, here labeled `arg_0`.

**Use the Strings window to locate the string `\cmd.exe /c` in the disassembly. Where is it located?**

All the way at the bottom of the list! The address is `xdoors_d:10095B34`.

**What is happening in the area of code that references `\cmd.exe /c`?**

This section of memory is referenced at location 0x100101D0. It's pushed to the stack as a parameter for `strcat` along with the local variable `CommandLine`. Based on some references to reading from a pipe, it looks like a command is being received from somewhere (probably a network connection) and then being prefixed with `\cmd.exe /c` to run the command.

**In the same area, at 0x100101C8, it looks like `dword_1008E5C4` is a global variable that helps decide which path to take. How does the malware set `dword_1008E5C4`?**

There are three references to `dword_1008E5C4`, but only one is a `mov` instruction. This is at 0x10001678. It's being set with the results of `sub_10003695`. This subroutine runs GetVersionExA and checks if dwPlatformId is 2, which [based on the documentation](https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-osversioninfoa) corresponds to VER_PLATFORM_WIN32_NT (Windows XP, 2000, etc.)

In short, it's checking if this is a 32-bit NT-based Windows. If not, it uses "command.exe" instead of "cmd.exe".

**A few hundred lines into the subroutine at 0x1000FF58, a series of comparisons use `memcmp` to compare strings. What happens if the string comparison to `robotwork` is successful (when `memcmp` returns 0)?**

This particular invocation happens at 0x10010452. If `memcmp` returns 0, it passes `s` (the sub's parameter, a socket) to `sub_100052A2`. This fetches the WorkTime from the registry, formats it into a string, and most likely sends it to the socket.

**What does the export PSLIST do?**

The first element of interest is the call to `sub_100036C3`. This checks the platform version: if the platform is not NT 32-bit, or the major version is less than 5, the export calls `sub_10006518`; otherwise it calls `sub_1000664C`.

The first subroutine, `sub_10006518`, collects information about all system processes, lists the processes' modules, and then writes them to a file. I am not sure where the filename is coming from, however, because the reference `[ebp-11Ch]` doesn't line up with a recognized variable. I suspect it's pointing to a section of the process's data to generate the filename. Perhaps further analysis will shed more light on this.

The second subroutine, `sub_1000664C`, also collects information about all system processes, but in this case it writes the output to the socket instead.