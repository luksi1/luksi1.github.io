---
layout: post
title: Zone transfer via nslookup
date: '2016-05-06 09:55:31'
---

You can do reverse and forward lookups via nslookup (Windows / Linux), dig or host (Linux), but searching for DNS aliases (CNAMEs) are bit a more complicated. The only way I know how to access these records is via the zone file. This is why being able to do a local zone transfer can be very convenient. 

<pre>$ host -l domain.name</pre>
"Host" works fine on linux, and "host" has been my go-to command when searching for CNAMES (DNS aliases). I have though now switched to Windows where nslookup is the de facto DNS tool.

nslookup is a bit more complicated, but works just as well.

<pre>nslookup 
> set type=any 
> ls -d domain.name > my.domains.txt
> exit</pre>

<pre>cat my.domains.txt | findstr my.domain.alias</pre>