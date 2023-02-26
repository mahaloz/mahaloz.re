---
title: "PwnAgent: A One-Click WAN-side RCE in Netgear RAX Routers"
layout: post
tags: [bug-hunting]
description: "A breakdown of a bug SEFCOM T0 and I exploited to achieve a WAN-side RCE in some Netgear RAX routers for pwn2own 2022. The bug is remote accessible command injection due to bad packet logging."
toc: true
---

Last December, my colleagues and I from [SEFCOM T0](https://sefcom.asu.edu/T0) competed in pwn2own 2022 where we [demonstrated an exploit to get RCE](https://twitter.com/mahal0z/status/1600322330970173441?s=20) in a Synology NAS. Although we were proud of this complicated exploit, we had a _much_ simpler, but more impactful, bug in another target that we never got to demo. That target was the [Netgear Nighthawk RAX30 Router](https://www.netgear.com/support/product/rax30.aspx), one of the latest and greatest models you can buy. Days before the competition, Netgear patched the bug in the _RAX30 Router_, the only model at pwn2own, eliminating our submission 😭. 

Netgear classified this [patch](https://kb.netgear.com/000065451/Security-Advisory-for-Multiple-Vulnerabilities-on-the-RAX30-PSV-2022-0028-PSV-2022-0073) as a LAN-side RCE; however, this bug can be easily exploited on WAN-side. Additionally, this bug _may still_ be present in the latest firmware of some of the **other** RAX models. In an exploration of some recent RAX versions, it's clear the code is shared. We believe Netgear knows about these bugs in the other firmware, but has delayed in pushing a fix for a while. Curious if your router is one of these affected models? Read on, we have a live demo running. 

Before talking about any of the _how_, let's talk about the impacts.

## Exploit Impacts
To exploit this bug on a victim router the attacker needs to do two things:
1. Setup a controlled website hosted on port 80 
2. Get the victim to visit the attacker's site from behind the victim's router 

This results in the attacker getting a RCE on the victim's router. For most RAX routers, this RCE is root, giving the attacker full control of your router. In the unfortunate scenario that you run nginx behind your vulnerable RAX router, this bug can be exploited with **0** interaction from you 🫢. 

For others newer to hacking routers, a root shell on a router (over remote) can allow an attacker to snoop on everything you visit (like a bad ISP 😬), read unencrypted traffic, mess with your DNS, and [do other nasty things](https://nordsecurity.com/blog/what-happens-when-router-is-hacked).

We don't know how many routers this affects, but, we do know this binary is shared by many. At the very least, if you are running a Netgear Nighthawk RAX30 Router that has not been updated since December of 2022, you are WAN-side pwnable. 

## Exploit Demo
We've created a fun (and safe 👌) way to know if your router is pwned. Visit this ☢️ [http://pwn.mahaloz.re](http://pwn.mahaloz.re) ☢️. If your router is vulnerable, it will shut your router off (and that is all). If you can refresh the page after visiting, it means your router is safe (hopefully). 

I also demoed the LAN-side exploit of this bug at [CactusCon 2023](https://youtu.be/-J8fGMt6UmE?t=23304) if you just want to watch a video :). Alright, let's get down to what this powerful bug is...

## The Bug
This bug was initially discovered by [@clasm](https://twitter.com/cl4sm), then reversed by me, and assisted by the rest of T0 team. For the rest of this blog, we will only reference the layout of the RAX30 firmware, as things may be slightly different in each model. The bug exists in a nonchalant binary in `/bin` called `puhttpsniff`. 

The `puhttpsniff` binary is responsible for logging tcp traffic on port 80. It will **only** log traffic that is leaving a local IP. This means the only traffic that gets recorded will be from devices already on the router. 

Every packet that meets these requirements will be logged. It will log the `UserAgent` and the IP. The logging is done using an [NFLOG](https://serverfault.com/questions/610989/linux-nflog-documentation-configuration-from-c) callback in user-space. That callback is implemented in a single function in `puhttpsniff`. Here is **all** the code of the callback function called when a packet meets the above requirements: 

### Function 0x10FF8
```c
char *__fastcall injection_func(const char *header_data, int header_len, int a3, const char *a4)
{
  char *result; // r0
  int v8[64]; // [sp+0h] [bp-318h] BYREF
  char v9[4]; // [sp+100h] [bp-218h] BYREF
  char v10[508]; // [sp+104h] [bp-214h] BYREF

  memset(v8, 0, sizeof(v8));
  *(_DWORD *)v9 = 0;
  result = (char *)memset(v10, 0, sizeof(v10));
  if ( header_len > 9 )
  {
    header_data[header_len] = 0;
    result = strstr(header_data, "User-Agent: ");
    if ( result )
    {
      _isoc99_sscanf(result + 12, "%255[^\r\n]", v8);
      sprintf(v9, "pudil -i %s \"%s\"", a4, (const char *)v8);
      return (char *)system(v9);
    }
  }
  return result;
}
```

The entire function is no more than 25 lines and has a glaring bug in the way `User-Agent` is used. The key lines are:
```c
_isoc99_sscanf(result + 12, "%255[^\r\n]", v8);
sprintf(v9, "pudil -i %s \"%s\"", a4, (const char *)v8);
return (char *)system(v9);
```

Yes, it's a textbook command injection. You only need to escape the quotes, which can be triggered like so:
```python
'User-Agent: "; {evil_command_here};"'
```

If you were paying attention earlier, you might recall that this should only be triggerable via LAN like the original Netgear report:
> "It will only log traffic that is leaving a local IP."

How could you, from the outside, control the UserAgent of a device on the inside of the router? Thanks to the web wisdom of [Adam Doupe](https://adamdoupe.com/), who was only passing by when we mentioned the bug, we found that Javascript can request arbitrary UserAgent responses from executors. Javascript, when loaded by your browser, can ask you to send certain requests that have server-requested UserAgents. 

Tested on the latest Firefox as of this post, you can request a custom UserAgent from visitors with the following JavaScript:
```html
<script>Object.defineProperty(navigator, 'userAgent', {{
        get: function () {{ return 'TEST'; }}
    }});
    const xhr = new XMLHttpRequest()
    xhr.open("GET", "/")
    xhr.setRequestHeader("User-Agent", '"; {evil_command_here};"');
    xhr.send()
</script>
```

With this in mind, the execution of the exploit goes something like this:

1. A victim get's on his router and is assigned an IP: `192.168.1.2`
2. The victim goes on Firefox and clicks link `malicious.site`
3. `malicious.site` send `192.168.1.2` the code of the site to render locally
4. `192.168.1.2` sees Javascript and executes the code
5. `192.168.1.2` sends an attacker controlled UserAgent back to `malicious.site`
6. In transit, the victim router records `192.168.1.2` UserAgent and gets pwned 

Take note that this also works for devices (or services) that just forward packets. In the case of an nginx server, which forwards packets, an attacker can simply send a custom UserAgent directly to the forwarding service. The service sends out the packet using a local IP which will be logged. 

This bug is very simple to understand, but, in practice, hard to automatically find. In the case of fuzzing, you would rapidly trigger this bug if you rapidly sent randomized packets to the router, but detecting that you triggered it is hard. If you did not know this was a command injection, it would look like nothing happened. The `system` called would go undetected if you did not already have full system emulation, AND, the `puhttpsniff` would not crash. 

I thought the simplicity of exploitation for this bug, and the strange place it occurred, warranted a fun bug name. I named it `PwnAgent`, since I really never thought a device would mess up parsing of the UserAgent. 

## Bug Discovery

At SEFCOM, we had a bunch of automated tooling and techniques to discover bugs that we used for our targets; however, this bug was ironically discovered manually while waiting for those tools.

Since we already had a testing shell on the device using `telnetenable`, clasm decided to look at which binaries we running by default on the router. He opened each of the binaries and did a quick search for use of `system`. One of the first binaries he opened was `puhttpsniff`. On **October 4th** he found the bug: 

![]({{ site.baseurl}}/assets/images/research/pwnagent/pwnagent_1.png)

Then by **October 5th**, we had a full end-to-end LAN exploit working after reversing how packets hit the callback function which we originally thought was un-triggerable:

![]({{ site.baseurl}}/assets/images/research/pwnagent/pwnagent_2.png)

It wasn't until later that month that we realized the exploit could be elevated to a WAN side attack. 

## Conclusion

I think there are a few interesting takeaways here. First, there are still _many_ trivial bugs that can be found with a fast decompiler and `grep` in embedded devices. Second, although this bug is easy to understand, automated analysis for embedded devices is still a _long_ way away from being able to detect a bug like this. Recall, to detect this bug you needed:
1. Full system emulation (hard)
2. Automated fuzz harnessing (hard)

At least publicly, we have neither of those things. To make the internet of things safer, we need to make moves in increasing automated analysis of firmware. If you know anyone with a Netgear router, please tell them to update their router to be on the safe side. 
