---
title: "WSL Crash on Windows 10"
date: 2021-09-01 21:00 -0700
categories: [Windows, VM]
tags: [Windows, WSL, VM, "Windows Subsystem for Linux", Hyper-V, CFG, "Control Flow Guard"]
excerpt: "When your `Windows Subsystem for Linux` VM crashes with no apparent error message, it could be due to a Windows security setting that blocks certain DLL calls."
# toc: true
classes: wide
---

This is one of those issues I ended up writing about because I had a hard time finding a solution for it as not many people seem to experience this under these specific circumstances. So I hope this makes somebody else's life easier in the future, too.

>**TL;DR:**<br><br>
>If `Control Flow Guard` causes your `Windows Subsystem for Linux` VM to crash by prohibiting certain DLL calls, turn it off for the Hyper-V related processes on your system.
{: .notice}

I recently attempted to install the latest version of `Windows Susbsystem for Linux (WSL)` and ran into multiple issues.

The first issues were related to having disabled all things Hyper-V on my system in the past as it was causing problems with VMware and VirtualBox. Now that I wanted to use WSL I had to re-enable everything which wasn't too difficult, just a little tedious as I had to find all the places where I needed to turn things back on (I'm looking at you, `bcedit`!...).

Once that was done I encountered the weird issue this post is about:

**My WSL VM kept terminating out of nowhere without any apparent error messages.**

There was no sign of any related activity in the console, the system just ended, long after `dmesg` received its last log entry:

![Screen_Shot_01_WSL_Console]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_01_WSL_Console.png" | relative_url }}){: .align-center}

Only when using the new `Windows Terminal` app I actually got an error displayed, I assume because internally it connects to the VM via its own remote connection handler:

>[process exited with code 1]
{: .notice--primary}

Not very helpful either.

When I started investigating the issue I noticed, that the times when the terminations happened did correlate in some way: these timestamps always shared the same minute and second, seemingly in a 30 minute pattern. In my case the terminations always occured at either HH:02:05 or HH:32:05.

![Screen_Shot_02_Event_Log_01]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_02_Event_Log_01.png" | relative_url }}){: .align-center}

The `System` log contained only the following error message:

>Error 7034: The Hyper-V Host Compute Service service terminated unexpectedly.  It has done this 8 time(s).
{: .notice--primary}

In the `Administrative Events` view - one that Windows assembles automatically to reduce noise for the system's administrator - it becomes more apparent that there is something scheduled on my system that keeps killing the VM. All related log entries shared the same time pattern I mentioned above.

![Screen_Shot_03_Event_Log_02]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_03_Event_Log_02.png" | relative_url }}){: .align-center}

The respective `Application Error` i.e. read:

>Error 1000: Faulting application name: vmcompute.exe, version: 10.0.19041.1151, time stamp: 0x9c99c33a<br>
>Faulting module name: ntdll.dll, version: 10.0.19041.1110, time stamp: 0xe7a22463<br>
>Exception code: 0xc0000409<br>
>...
{: .notice--primary}

There was another entry with the same timestamp in the normal `Application` log that wasn't populated in the `Administrative Event` view as it is of the `Info` category, and it had this:

>Info 1001: Fault bucket 1219256380472107854, type 5<br>
>Event Name: BEX64<br>
>Response: Not available
{: .notice--primary}

So what these log entries tell me is that the application running the WSL VM crashed due to a failed call to a system DLL. The time pattern is a hint for something the systems is scheduled to run every 30 minutes. I did some testing to verify this observation and was able to confirm the time pattern:

- starting the VM right before the clock hit a time matching this specific pattern lead to a crash at said time
- starting it anytime after that time kept it running until the next match for that time pattern, so up to almost 30 minutes

I didn't know yet why that happened, but I had enough information to get searching the internet for more on that.

It turned out that Windows 10 and above have a security feature that's called `Exploit Protection`. This is meant to block calls from programs to malicious libraries, but why it categorizes Microsoft's own system components as threat, that I don't know. Since this doesn't seem to be a very common issue it likely has to do with how this particular system was configured.

However, here now is the information you need to fix this, in case it happens on your system, too.

First, open the Windows `Settings` app and go to the `Security` section.

![Screen_Shot_04_Windows_Security]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_04_Windows_Security.png" | relative_url }}){: .align-center}

Then click `App & browser control` and scroll down to `Exploit protection`. Click on `Exploit protection settings`.

![Screen_Shot_05_App_browser_control]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_05_App_browser_control.png" | relative_url }}){: .align-center}

Now you'll see two tabs, switch to the one named `Program settings` and scroll all the way down to where the following two binaries are listed:

- `C:\Windows\System32\vmcompute.exe`
- `C:\Windows\System32\vmwp.exe`

![Screen_Shot_06_Exploit_protection]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_06_Exploit_protection.png" | relative_url }}){: .align-center}

Click `Edit` for both of them and turn off the `Control flow guard (CFG)` setting.

![Screen_Shot_07_CFG_01]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_07_CFG_01.png" | relative_url }}){: .align-center}

![Screen_Shot_08_CFG_02]({{ "/assets/images/2021-09-01-WSL-crash-on-Windows-10/Screen_Shot_08_CFG_02.png" | relative_url }}){: .align-center}

Now restart the VM to apply the changes. If it doesn't work right away verify these two files have no running instances before starting the VM again.

A note about security:

I don't know exactly about the security implication of turning off this setting, but keep in mind that this is a `Windows Defender` integrated security feature that is meant to prevent memory corruption, e.g. in the case of a ransomware trying to cause a buffer overflow. So change these settings on your own risk and be careful about what software you run on your computer and inside your VM's. To learn more about `CFG` go to the [Microsoft documentation for Control Flow Guard](https://docs.microsoft.com/en-us/windows/win32/secbp/control-flow-guard).
