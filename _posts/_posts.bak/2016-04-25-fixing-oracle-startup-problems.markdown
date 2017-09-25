---
layout: post
title: Fixing Oracle startup problems
date: '2016-04-25 22:58:17'
---

It happens from time to time when starting up your Oracle instance that you set a value you shouldn't have. Oops. Nothing starts.

<pre>SQL&gt; startup
ORA-00823: Specified value of sga_target greater than sga_max_size</pre>

<br>You simply need to create a pfile from your current spfile, edit your pfile and then mount your instance with that pfile.</br>

<pre>SQL&gt; create pfile='/tmp/foo.pfile' from spfile;

File created.

Edit '/tmp/foo.pfile' and then mount our database with this pfile.

SQL&gt; startup nomount pfile=/tmp/foo.pfile
ORACLE instance started.

Total System Global Area 1.7103E+10 bytes
Fixed Size                  2267984 bytes
Variable Size            2046821552 bytes
Database Buffers         1.5032E+10 bytes
Redo Buffers               21688320 bytes</pre>

<br>And then re-create our spfile from our pfile.</br>

<pre>SQL&gt; create spfile from pfile='/tmp/foo.pfile';

File created.</pre>