---
title: "I'll Crash your Netflix TV: finding CVE-2019-10028"
layout: post
tags: [research, bug-hunting, pwn, fuzzing]
description: "Finding an out-of-bounds read on a Netflix-DIAL server with Mayhem"
toc: true
---

## Introduction 
This month, while testing Mayhem on various open-source projects, one open-source project had some fruitful Mayhem results. The Netflix Dial Server
located in the Netflix Github repository had a remote out-of-bounds read which
Mayhem discovered. 

Since many other fuzz testers are unable to fuzz network communication efficiently, most webserver-like programs are full of bugs that Mayhem can trigger. 

## Background 
In early 2013 Netflix was working on a project called Discovery And Launch,
better known as DIAL. The DIAL server was one of the first prototypes for streaming Netflix and YouTube directly from your smartphone to your television.
The DIAL server would come preinstalled on most TVs of the time. If this all
sounds too familiar, it's because Google created a device called Google
Chromecast that does the same thing. Google Chromecast originally used DIAL as
its means of communication but eventually switched to mDNS--thus began the end
of DIAL's usage. Although DIAL stopped shipping with new televisions, it's not clear that DIAL is not purged from modern Netflix and YouTube applications.
After a small investigation, we found that the DIAL server could still
communicate with an iPhone--making this an interesting target. 

![iPhone POC]({{ site.baseurl}}/assets/images/research/netflix-writeup-2019/iphone-netflix-poc.jpeg)

## Recon
Looking at the dial-reference readme, it says: "The DIAL client uses CURL to
send HTTP request commands to the DIAL server," which is right up Mayhem's alley
of operation. Quickly reversing some of the dial-server code shows that the
server listens over a constant IP Address and port.

### dial_server.c
```c
42 // TODO: Partners should define this port
43 #define DIAL_PORT (56789)
44 #define DIAL_DATA_SIZE (8*1024)
45
46 static const char *gLocalhost = "127.0.0.1";
47
```

The main component of the DIAL server is the Mongoose server, which is based on
the original Mongoose embedded server released around the same time as DIAL.
This code is significantly modified to be as compact as possible.

### mongoose.h
```c
22 // NOTE: This is a SEVERELY stripped down version of mongoose [...] 
```

Weighing in at about 960 lines, the mongoose.c file had the potential to be
buggy, so we decided to finally start Mayhem on the DIAL Server.

## Harnessing
Harnessing is usually one of the harder tasks when fuzzing webservers, but
luckily the engineers at ForAllSecure have had their fair share of fuzzing.
Simply running the Mayhem packaging system on the DIAL Server binary collected
all the linked libraries and placed them into a well formatted file. The last
part of harnessing was simply specifying what networking protocol (TCP or UDP),
what port, and what IP to fuzz on--presto, we have a fuzzer.

## Fuzzing Performance
Mayhem did surprisingly well on the binary without modification to the source
code or customized compilation. Executions clocked in at around 1400 executions
per second, which is not amazing but still awesome. The server explored branches
relatively fast and found a crash within the first 24 hours. 

## Triaging Rouge Requests
The DIAL server crashed with a SIGSEGV showing a backtrace culprit of 
`discard_current_request_from_buffer` function. After spending some time tracking
the passing of bytes and requests, it was clear that the problem was coming from the
`mongoose.c` code. 

```
f 0     7ffff774fc3c __memmove_avx_unaligned_erms+364
f 1     55555555b605 discard_current_request_from_buffer+252
f 2     55555555b89f process_new_connection+663
f 3     55555555bb79 worker_thread+284
f 4     7ffff7bbd6db start_thread+219
```

When the dial-server program is launched it spawns multiple threads and makes a
call to `mongoose.c` to process the incoming connections.

The function `process_new_connection` is called to process the current request. The
requested `content-length`, specified in the GET request, is placed into the variable
`conn->request_len` and later used in a `memmove` call for an address offset.

The vulnerability is found in the check on the request length. On line 734 an
assert is called: `assert(conn->data_len >= conn->request_len)`. Though the assert
checks for `conn->request_len` to be less than the allocated `data_len`, the assert
does not check if the `conn->request_len` is negative. No check on the value
allows this value to be passed to the `discard_current_request_from_buffer`
function as the new `content_len`.

```c
734   assert(conn->data_len >= conn->request_len);
735   if (conn->request_len == 0 && conn->data_len == conn->buf_size) {
736     mg_send_http_error(conn, 413, "Request Too Large", "");
737     return;
738   } if (conn->request_len <= 0) {
739     return;  // Remote end closed the connection
740   }
```

Once `content_len` makes it to the discard function, another check is made
on line 712 to check the `content_len` is smaller than the buffered size:
`else if (conn->content_len < (int64_t) buffered_len)`
which will pass and do:
`body_len = (int) conn->content_len;`
This allows a large 64-bit negative value (larger than 32 bit), to be converted
into 32 bits, which allows the negativeness to be dropped. So a value passed of
length: `-13377777777777 (0xfffff3d53e4ec38f)`, gets converted to
1045349263--allowing an arbitrary length read of a memory location.

```c
712     else if (conn->content_len < (int64_t) buffered_len) 
713     body_len = (int) conn->content_len;
```

If the passed size is of a certain size, it will cause a `memmove` of a non-readable
memory location, which will crash the program.

## Crashing Example
When sent to the server, it causes a SIGSEGV crash. 
```
GET /foo HTTP/1.1
Content-Length:-12377777775679
```

## Simple Fix, Complex Benefit
Since the bug is found in the length check, a fix could be as simple as changing:
`assert(conn->data_len >= conn->request_len);`
to:
`assert(conn->data_len >= conn->request_len && conn->request_len > 1);`

## Responsible Disclosure
Here at ForAllSecure we take responsible disclosure very seriously because one bad-timed bug can cause a lifetime of ill impact. Before making this post we
contacted Netflix and informed them of the bug and the simple way to fix it. The
bug was publicly fixed on the Netflix GitHub with the commit [d1b1aa26](https://github.com/Netflix/dial-reference/commit/d1b1aa2636f89df95e57aa0c68836ce8d52f4638).
Credit for the reference was also published in their reference repository at
[nflx-2019-003.md](https://github.com/Netflix/security-bulletins/blob/master/advisories/nflx-2019-003.md).
After confirming the fix had been pushed, we created this blog post.

## Credit Where Credit is Due
Finding and triaging this bug would have not been possible without the hard work that engineers put into making Mayhem awesome and without my fellow coworker
Paul Emge, who spent as much time as I did on this bug. 

## Other Sources
The discovery of this bug is also mentioned in [Security
Boulevard](https://securityboulevard.com/2019/09/forallsecure-uncovers-vulnerability-in-netflix-dial-software/)
, [ForAllSecure
Blog](https://blog.forallsecure.com/forallsecure-uncovers-vulnerability-in-netflix-dial-software),
and
[Axios](https://www.axios.com/netflix-chromecast-bug-crash-television-c4f0962b-0346-4a16-aafc-4c512e76d34b.html). 

The CVE is listed as
[CVE-2019-10028](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-10028).



