---
title: "How to extract Python source code from Py2App packed Mach-O Binaries"
date: 2021-03-06
categories: [malware-analysis, reverse-engineering]
tags: [macos, malware, reverse-engineering, web3, python]
description: "I got many requests after my last tweet on the discovery of a backdoored Electrum wallet, that was notarized by Apple !"
toc: true
---

*I got many requests after my last tweet on the discovery of a backdoored Electrum wallet, that was notarized by Apple !*

![Photo by Raul Cacho Oses on Unsplash](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-01.png)
_Photo by Raul Cacho Oses on Unsplash_

I got many requests after my last tweet on the discovery of a backdoored Electrum wallet, that was notarized by Apple !


> **Tweet**: [https://twitter.com/lordx64/status/1367491506471370763](https://twitter.com/lordx64/status/1367491506471370763)

The requests were about how I was able to extract the python sourcecode from a Py2App packaged Mach-O file and discover the backdoor code.

Here’s the hash to use for this morning quick tutorial : 313efd27555f2d024e56d88a31ffda045e0e0d13db868867e416f0330e3ba773 electrum-4.0.9.dmg

Below are the steps I followed


#### Step 1

- Mount the .dmg file, and locate the packed mach-o file we want to extract the source code from :


```
hdiutil mount electrum-4.0.9.dmg
```


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-02.png)


#### Step 2

Sweet! now let’s locate the Mach-o **run_electrum** file, packed with py2App

For this wallet , it is usually under


```
Electrum/Electrum.app/Content/MacOS/run_electrum
```


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-03.png)


#### step 3

Use **Binwalk** :)

ReFirmLabs/binwalkBinwalk supports Python 2.7 - 3.x. Although most systems have Python2.7 set as their default Python interpreter…github.com**Binwalk** is the go to tool for firmware analysis :) but let’s keep it simple, and run it on this **run_electrum** mach-o file, you will notice a couple of zlib headers are detected and that’s where the python compiled code is hiding:


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-04.png)

Sweet ! now we could use the same command, and we add the -e switch to Binwalk to automatically extract the zlibs for us. **Binwalk** will create a directory called **_run_electrum.extracted**, where we can find all the extracted python compiled code :


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-05.png)


#### Step 4

Now that we extracted Python compile code, let’s try to find the original **run_electrum.pyc** file ?


```
grep -Rni --color "run_electrum" *
```

Gave us one and only hit the file D5E9 so we will rename this file to **run_electrum.pyc** for now:


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-06.png)

This file looks like random data, but you will probably see it is a python compiled code :)


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-07.png)

Only that it doesn’t contain a valid Python 3.7 header needed for decompyl3 to work.


#### Step 5

get **decompyle3** at [https://github.com/rocky/python-decompile3](https://github.com/rocky/python-decompile3)

if you run **decompyle3 **on this **run_electrum.pyc **it won’t work and will complain about the header as expected:


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-08.png)

in this case let’s give **decompyle3 **a valid python 3.7 header :)


#### Step 6

Let’s patch the **run_electrum.pyc** header.

> Note: Normally I should use [Hiew](http://www.hiew.ru/) to pay respect to the reverse engineers out there, but man it’s 7 am, and I am using my mac right now..

So I will use [Hex Fiend](http://hexfiend.com/), which is a GUI tool to patch binaries. let’s open **run_electrum.pyc** with Hex Fiend:

And add the following header :


```
420D0D0A 00000000 00000000 00000000
```


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-09.png)


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-10.png)

run **decompyle3** again and see the magic happens:


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-11.png)

This will create a **run_electrum.py** for you, you can now open it with your favorite text editor, and enjoy python:


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-12.png)


#### Step 6 : bonus

Where to find the backdoor code ? you can repeat the steps above, on the compiled object **1D6CF** file, this file is the python compiled object of the original file** electrum/util.py** where the backdoor was inserted. If you open this file after decompilation you will be greeted with free Firestore credentials, and the prefix_log() function that was inserted to exfiltrate the wallets and the wallet passwords ..


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-13.png)


![image](/assets/images/posts/how-to-extract-python-source-code-from-py2app-packed-mach-o/img-14.png)

If you liked this quick tutorial or similar discoveries, please show me some love and follow me on Twitter at [@lordx64](https://twitter.com/lordx64?lang=fr)

Have a great weekend !
