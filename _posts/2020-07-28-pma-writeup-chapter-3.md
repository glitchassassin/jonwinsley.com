---
layout:     post
title:      PMA Labs Writeup - Chapter 3
date:       2020-02-21 12:00:00
author:     Jon Winsley
comments:   true
summary:    My analysis of the labs in Chapter 3 of "Practical Malware Analysis".
categories: reverse-engineering
---

[Practical Malware Analysis](https://practicalmalwareanalysis.com/) is still a handbook for aspiring malware analysts, and while I've dabbled in the subject before, I've decided to work through the book for a better hands-on grasp of malware reverse engineering. Needless to say, this writeup will contain spoilers.

# Chapter 3: Basic Dynamic Analysis

I skipped the writeup for chapter 1's labs, which were more or less a matter of uploading samples to VirusTotal and reading the outputs. Chapter 2 dealt with setting up a virtual environment, but when I started in on the labs for Chapter 3 I realized I needed something very specific. So we'll deal with that below.

Chapter 3 starts to get into the meat of things: some basic dynamic analysis. In simple terms, running the malware to see what it does. There are a couple tools we'll use to watch how the malware interacts with the system, the disk drive, and the network.

## Lab Setup

I started with a Windows 10 VM in Hyper-V, installing the [flare-vm distribution](https://github.com/fireeye/flare-vm) from FireEye. Because my VM's disk wasn't stored on an SSD, this install took forever, and when it completed I discovered the malware samples for Lab 3 wouldn't even run in Windows 10. Chalk that one up to experience!

Instead, I found [a Windows XP VM image](https://helpdeskgeek.com/virtualization/how-to-set-up-a-windows-xp-virtual-machine-for-free/) from Microsoft, with a little effort, and set that up. I had to use a legacy network adapter to get connected to the internet.

Now I had a new conundrum, as the version of Internet Explorer (IE 6.0!) doesn't know how to speak modern ciphers, so it's unable to access much of the modern web. I changed the internet options to enable TLS 1.0, then downloaded Chrome; after running Chrome with --ignore-certificate-errors I was able to get some semblance of functionality. Some sites, such as SourceForge, I still couldn't reach, but I was able to at least get the tools referenced in the book.

After installing all the static and basic dynamic analysis tools, I set up an isolated private switch in Hyper-V to enable network without exposing the machine to the internet. I set up a static IP in the VM, shut it down, and took a snapshot.

Whew. Now we can finally get to the analysis.

## Lab 03-01

File hash: [eb84360ca4e33b8bb60df47ab5ce962501ef3420bc7aab90655fd507d2ffcedd](https://www.virustotal.com/gui/file/eb84360ca4e33b8bb60df47ab5ce962501ef3420bc7aab90655fd507d2ffcedd/detection)

