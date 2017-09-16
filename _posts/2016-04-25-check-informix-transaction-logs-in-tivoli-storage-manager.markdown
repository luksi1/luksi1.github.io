---
layout: post
title: Check Informix transaction logs in Tivoli Storage Manager
date: '2016-04-25 22:47:04'
---

You may want to find an Informix transaction log, or verify a transaction log, or logs, that have been sent to TSM. You can do this from the client.
<pre>dsmc query backup -inactive "{filespace name}/{filespace name}/{instance number}/*"</pre>

<br>You can retrieve the file space name with "dsmc q fi". The file space name is the same as the IPC name of your instance.  The instance number you can find under the SERVERNUM parameter via "onstat -c".</br>

An example:
<pre>dsmc query backup -inactive "/ipcmypak1/ipcmypak1/0/*"</pre>

&nbsp;