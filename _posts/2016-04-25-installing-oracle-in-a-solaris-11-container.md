---
layout: post
title: Installing Oracle in a Solaris 11 container
date: '2016-04-25 22:42:46'
---

Solaris containers (or zones) make a perfect platform for Oracle databases. Containers are now a very mature technology. They use ZFS as their underlying file system. Plus, and here's the kicker, containers are considered by Oracle as <a href="http://www.oracle.com/technetwork/server-storage/solaris11/technologies/os-zones-hard-partitioning-2347187.pdf">"hard partitions"</a>. This means that you can pin cores to your container and you only have to pay for that core! In the future, you could scale your infrastructure without changing the physical infrastructure!

You can read more about resource pools in Solaris <a href="https://docs.oracle.com/cd/E19683-01/817-1592/rmpool.task-1/index.html">here</a>. Brenden Gregg also has a nice blogg entry with examples <a href="http://www.brendangregg.com/zones.html">here</a>.

##Configuring
Install the following packages to get X working:
<pre>pkg install pkg://solaris/library/motif
pkg install pkg://solaris/compatibility/packages/SUNWxwplt</pre>

<br>If you log out and log in again, your DISPLAY should be set by your SSH session and the runInstaller script should work.</br>

You will also need to install the following packages according to the Oracle Database for Solaris 11 prerequisites.
<pre>pkg install pkg://solaris/developer/build/make
pkg install pkg://solaris/system/xopen/xcu4
pkg install pkg://solaris/x11/diagnostic/x11-info-clients
pkg install pkg://solaris/compress/unzip
pkg install pkg://solaris/developer/assembler
pkg install pkg://solaris/system/dtrace</pre>

<br>Shared memory allocation appears to now work out-of-the-box on Solaris 11, allowing a container to allocate as much shared-memory as it likes (if not otherwise limited via <a href="https://docs.oracle.com/cd/E36784_01/html/E36871/zonecfg-1m.html">zonecfg</a>).</br>
<pre># prtconf | grep Mem
prtconf: devinfo facility not available
Memory size: 262016 Megabytes</pre>

<br>This is also confirmed <a href="https://docs.oracle.com/cd/E11882_01/install.112/e24349/toc.htm#CDEEHBFC">here</a>.</br>

##Solaris 10
On Solaris 10, you will need to allocate a pool, otherwise you will run into shared-memory allocation problems.

I haven't tested my old documents for awhile now, and <a href="https://blogs.oracle.com/mandalika/entry/oracle_on_solaris_10_fixing">this blog</a> entry from Oracle is certainly more comprehensive, but something along these lines should work.

/etc/system should look like this:
<pre>set max_nprocs=20000
set maxuprc=16384
set shmsys:shminfo_shmmax=4294967295</pre>

<br>The documents will tell you to use the resource control facility, but /etc/system still works on Solaris 10.</br>

And then, add your project:
<pre>projadd -U oracle user.oracle
projmod -a -K "project.max-shm-memory=(priv,65gb,deny)" user.oracle</pre>
