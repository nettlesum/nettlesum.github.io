---
layout: post
title:  "Basic packet analysis for beginner security analysts"
date:   2022-10-05 12:04:00
categories: wireshark
---

# Basic packet analysis for beginner security analysts

I thought I’d try and brush up on my packet analysis skills by looking at an older exercise from Brad Duncan’s Malware Traffic Analysis blog. Specifically, we’re looking at [a traffic analysis exercise from the 16th of November, 2014](https://www.malware-traffic-analysis.net/2014/11/16/index.html) (so it’s quite dated now).

Before looking to answer the questions Brad put together I want to try and understand what’s going on in the packet capture.

## Initial high-level analysis

For me, the best place to start is dropping the packet capture into VirusTotal and getting an idea of whether any alerts were triggered. You can find these on the Details page. In this case, we can see a number of alerts from both Snort and Surikata. 

These Snort alerts in particular stand out:

> **A Network Trojan was detected**
> 
>  - FILE-JAVA Oracle Java obfuscated jar file download attempt [25562]
>  - EXPLOIT-KIT Multiple exploit kit jar file download attempt [27816]
>  - EXPLOIT-KIT Goon/Infinity/Rig exploit kit encrypted binary download [30934]
>  - EXPLOIT-KIT Goon/Infinity/Rig exploit kit outbound uri    structure [30936]

Searching these in [Snort's rule documentation](https://www.snort.org/rule_docs/) can provide us with further information on each. Through these alerts alone we have a pretty good idea that the machine has likely been infected using the RIG exploit kit. This gives us a good place to start in our analysis.

## Diving in with Wireshark

From our high-level analysis we know that an exploit kit was likely used to infect the machine, so we can start by looking for the files that were downloaded.

To do this we can Export HTTP objects in Wireshark via **File** > **Export Objects** > **HTTP** and sort by the **Content Type** column to bring application content to the top of the list. 

![A list of exported HTTP objects in Wireshark, including Java, executable and flash files of note.](/exports.png)

Based on our alerts from Snort, we're particularly curious about the Java file here, but we'll take a sample of each to make sure we're covering all of our bases. 

In this case, we’ll save a sample of the .jar (Java), .exe (executable) and the .swf (Flash) files to hash and search. Once saved, you can use PowerShell to quickly grab an MD5 hash for each of the files. I’ve renamed the files here for ease of reading:

```*get-filehash -a MD5 '.\exe.exe' ; get-filehash -a MD5 '.\jar.jar' ; get-filehash -a MD5 '.\swf.swf'*```

The hashes that we get for each are:

>**Java File:** 1E34FDEBBF655CEBEA78B45E43520DDF
>**Flash File:** 7B3BAA7D6BB3720F369219789E38D6AB
>**Executable File:** D276C86DCDBCDB6B74EE02496BC90D98

Upon checking each of these hashes on VirusTotal we find that the Java and Flash files are known to be malicious, while the executable file does not raise any flags.

We now know that the malicious files were delivered from the **stand[.]trustandprobaterealty[.]com** domain. Reviewing the HTTP traffic in the capture, we can find the IP for that domain, which is **37.200.69.143**. If we open the HTTP section in the packet details section for this entry we can follow the referrers back. The referrer for this entry is **24corp-shop[.]com**, checking it's referrer, we find **www[.]ciniholland[.]nl**.

We can identify the source address of these requests as **172.16.165.165**. If we filter for dhcp traffic and open the DHCP Request we can find the MAC Address and host name for the source address, in this case **f0:19:af:02:9b:f1** and **K34EN6W3N-PC** respectively.  

![The MAC address and host name of the infected machine as seen in Wireshark.](/macandhost.png)

We now have an understanding of what happened, so we can put together an incident report.

## Incident report

>### Executive Summary
>
>On 2014-11-16 at approximately 02:12 UTC a Windows virtual machine was infected by an attacker using the RIG Exploit Kit. 
>
>### Details of Infected Host
>
>- MAC address: f0:19:af:02:9b:f1
>- IP address: 172.16.165.165
>- Host name: K34EN6W3N-PC
>
>### Indicators of Compromise
>
>We were able to identify multiple indiactors of compromise, including details of the domain where the malicious files were downloaded from, the file hashes themselves, and the referring sites and relevant IP addresses.
>
>**Details for RIG EK Download(s):**
>
>- 37.200.69.143
>- stand[.]trustandprobaterealty[.]com
>
>**Sites/Referrers**
>
>- 82.150.140.30 / www[.]ciniholland[.]nl
>- 188.225.73.100 / 24corp-shop[.]com
>
>### Malicious Files 
>
>We found two malicious files in our investigation, a Java file and a Flash file. Details for both can be found below.
>
>**Java File**
>- MD5: 1E34FDEBBF655CEBEA78B45E43520DDF
>- File size 10.36 KB (10606 bytes)
>- File type: JAR
>
>**Flash File**
>- MD5: 7B3BAA7D6BB3720F369219789E38D6AB
>- File size: 8.03 KB (8227 bytes)
>- File type:	Flash

## Answering the exercise questions

Our high-level analysis and deep dive in Wireshark has given us every piece of information needed in order to answer Brad's level one questions for this exercise:

**1) What is the IP address of the Windows VM that gets infected?**

172.16.165.165

**2) What is the host name of the Windows VM that gets infected?**

K34EN6W3N-PC

**3) What is the MAC address of the infected VM?**

f0:19:af:02:9b:f1

**4) What is the IP address of the compromised web site?**

82.150.140.30

**5) What is the domain name of the compromised web site?**

www[.]ciniholland[.]nl

**6) What is the IP address that delivered the exploit kit and malware?**

37.200.69.143

**7) What is the domain name that delivered the exploit kit and malware?**

stand[.]trustandprobaterealty[.]com

# Conclusion

I hope that you found something useful in this post and write up. I'll be sure to update this post as my process for analysing these types of files improves and grows in the future. 
