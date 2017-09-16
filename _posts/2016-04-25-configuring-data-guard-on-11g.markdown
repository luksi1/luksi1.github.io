---
layout: post
title: Configuring Data Guard on 11g
date: '2016-04-25 22:52:40'
---

Data Guard is Oracle's disaster recovery option for systems that require an extremely high uptime and low mean time to recovery (MTTR).

######Environment
Primary host: node1
Primary database: db1

Standby host: node2
Standby database: db2

######Configuring
Set up both nodes with the same tablespaces. Be sure that each database is configured in tnsnames.ora on each node. Add the option UR=A to allow access to listeners in restricted or blocked status.

Primary (node1):
<pre>DB2 =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = node2.domain.com)(PORT = 1521))
(CONNECT_DATA =
(SERVER = DEDICATED)
(SERVICE_NAME = DB2)
<strong>(UR = A)</strong>
)
)</pre>

<br>Standby (node2):</br>

<pre>DB1 =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = node1.domain.com)(PORT = 1521))
(CONNECT_DATA =
(SERVER = DEDICATED)
(SERVICE_NAME = DB1)
<strong>(UR = A)</strong>
)
)</pre>

<br>Test accessibility from each node.</br>

<pre>node1:~$ tnsping db1
node2:~$ tnsping db2</pre>

<br>Test permissions.</br>

On the primary.
<pre>node1:~$ sqlplus sys/test@db2 as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Wed Feb 3 21:18:19 2016

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL&gt; show parameters db_unique_name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_unique_name                       string      db2
SQL&gt; select database_role from v$database;
select * from v$database
*
ERROR at line 1:
<strong>ORA-01507: database not mounted</strong></pre>

<br>On the standby.</br>

<pre>node2:~$ sqlplus sys/test@db1 as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Wed Feb 3 21:24:45 2016

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL&gt; show parameters db_unique_name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_unique_name                       string      db1
SQL&gt; select database_role from v$database;

DATABASE_ROLE
----------------
PRIMARY</pre>

######Static listener
You must configure a static listener when using this method! When connecting to the standby database from the primary, the duplication process will require you to bounce the instance. When this is done, if you did not configure a static listener, the instance would de-register itself from the listener and you will get bounced from your connection. To avoid this, configure a static listener so the instance cannot de-register itself.

Primary
<pre>node1:~$ cat $ORACLE_HOME/network/admin/listener.ora
LISTENER =
(DESCRIPTION =
(ADDRESS_LIST =
(ADDRESS = (PROTOCOL = TCP)(HOST = node1.domain.com)(PORT = 1521))
)
)

SID_LIST_LISTENER =
(SID_LIST =
(SID_DESC =
(SID_NAME = db1)
(ORACLE_HOME = /opt/oracle/app/oracle/product/11.2.0/dbhome_1)
)
)</pre>
<br>Standby.</br>
<pre>node2:~$ cat $ORACLE_HOME/network/admin/listener.ora
LISTENER =
(DESCRIPTION =
(ADDRESS_LIST =
(ADDRESS = (PROTOCOL = TCP)(HOST = node2.domain.com)(PORT = 1521))
)
)

SID_LIST_LISTENER =
(SID_LIST =
(SID_DESC =
(SID_NAME = db1)
(ORACLE_HOME = /opt/oracle/app/oracle/product/11.2.0/dbhome_1)
)
)

ADR_BASE_LISTENER = /opt/oracle/app/oracle</pre>

<br />
######Configure standby logs
View your logs on the primary database.
<pre>SQL&gt; select a.group#, a.member, b.bytes from v$logfile a, v$log b  where a.group#=b.group#;</pre>

<br>And then create at least 3 standby logs that reflect these logs. Use the next group number available. And use the same size. Create these logs on both nodes!</br>

<pre>ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 ('/opt/oracle/app/oracle/oradata/mydb/redo04.log') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 ('/opt/oracle/app/oracle/oradata/mydb/redo05.log') SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 ('/opt/oracle/app/oracle/oradata/mydb/redo06.log') SIZE 500M;</pre>
<br />
######Configure standby server
On the standby server (node2), shutdown your standby database. Then create a pfile and startup the database in "nomount" with our pfile.
<pre>node2:~$ pwd
/opt/oracle

node2:~$ cat dg.ora
db_name=db1
db_unique_name=db2

node2:~$ sqlplus "/as sysdba"

SQL*Plus: Release 11.2.0.4.0 Production on Wed Feb 3 11:32:51 2016

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL&gt; startup nomount pfile=/opt/oracle/dg.ora</pre>

<br />
######Duplicate the primary database
Right now, your control files will not match. The paths will be different. You need to convert your spfile.

Create a pfile from our spfile on the primary.
<pre>SQL&gt; create pfile='/tmp/pfile.txt' from spfile;</pre>
<br>Make a list of the entries, which would need to be converted. Ignore db_name, as these are the same on both databases.</br>
<pre>node1~$ cat /tmp/pfile.txt
<strong>*.audit_file_dest='/opt/oracle/app/oracle/admin/db1/adump'</strong>
*.audit_trail='db'
*.compatible='11.2.0.4.0'
<strong>*.control_files='/opt/oracle/app/oracle/oradata/db1/control01.ctl','/backup/oracle/recovery_area/db1/control02.ctl'</strong>
*.db_block_size=8192
*.db_domain=''
<del>*.db_name='db1'</del>
*.db_recovery_file_dest='/backup/oracle/recovery_area'
*.db_recovery_file_dest_size=8589934592
<strong>*.db_unique_name='db1'</strong>
*.diagnostic_dest='/opt/oracle/app/oracle'
<strong>*.fal_client='db1'</strong>
<strong>*.fal_server='db2'</strong>
*.log_archive_config='DG_CONFIG=(db1,db2)'
<strong>*.log_archive_dest_1='LOCATION=/backup/db1'</strong>
<strong>*.log_archive_dest_2='service=db1 lgwr async valid_for=(online_logfiles, primary_role) db_unique_name=db2'</strong>
*.log_archive_dest_state_1='enable'
*.log_archive_dest_state_2='enable'
*.open_cursors=300
*.pga_aggregate_target=23177723904
*.processes=150
*.remote_login_passwordfile='EXCLUSIVE'
*.sga_target=4293918720
*.undo_tablespace='UNDOTBS1'</pre>
<br>Now create an .rcv file, /opt/oracle/scripts/standby.rcv, on the primary node (node1) with the following content. We're going to use this script to run RMAN from the primary, to duplicate our database on to the standby.</br>

<pre>run {
allocate channel d1 type disk;
allocate channel d2 type disk;
allocate auxiliary channel stby type disk;
duplicate target database for standby from active database
dorecover
spfile
parameter_value_convert 'db1','db2'
set db_unique_name='db2'
set db_file_name_convert='/opt/oracle/app/oracle/oradata/db1/','/opt/oracle/app/oracle/oradata/db2/'
set control_files='/opt/oracle/app/oracle/oradata/db2/control01.ctl','/backup/oracle/recovery_area/db2/control02.ctl'
set log_archive_max_processes='10'
set fal_client='db2'
set fal_server='db1'
set standby_file_management='AUTO'
set log_archive_dest_1='LOCATION=/backup/db2'
set log_archive_dest_2='service=db1 lgwr async valid_for=(online_logfiles,primary_role) db_unique_name=db1'
;
}</pre>

<br>And then run it:</br>

<pre>$ rman target sys/test@db1 auxiliary sys/test@db2

Recovery Manager: Release 11.2.0.4.0 - Production on Thu Feb 4 15:18:53 2016

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

connected to target database: DB1 (DBID=4002159261)
connected to auxiliary database: DB1 (not mounted)

RMAN&gt; @backup.standby.rcv

RMAN&gt; run {
2&gt;  allocate channel d1 type disk;
3&gt;  allocate channel d2 type disk;
4&gt;  allocate auxiliary channel stby type disk;
5&gt;  duplicate target database for standby from active database
6&gt;  dorecover
7&gt;  spfile
8&gt;   parameter_value_convert 'db1','db2'
9&gt;   set db_unique_name='db2'
10&gt;   set db_file_name_convert='/opt/oracle/app/oracle/oradata/db1/','/opt/oracle/app/oracle/oradata/db2/'
11&gt;   set control_files='/opt/oracle/app/oracle/oradata/db2/control01.ctl','/backup/oracle/recovery_area/db2/control02.ctl'
12&gt;   set log_archive_max_processes='10'
13&gt;   set fal_client='db2'
14&gt;   set fal_server='db1'
15&gt;   set standby_file_management='AUTO'
16&gt;   set log_archive_dest_1='LOCATION=/backup/db2'
17&gt;   set log_archive_dest_2='service=db1 lgwr async valid_for=(online_logfiles,primary_role) db_unique_name=db1'
18&gt; ;
19&gt; }
using target database control file instead of recovery catalog
allocated channel: d1
channel d1: SID=185 device type=DISK

allocated channel: d2
channel d2: SID=193 device type=DISK

allocated channel: stby
channel stby: SID=249 device type=DISK

Starting Duplicate Db at 04-FEB-16

...

starting media recovery

archived log for thread 1 with sequence 3511 is already on disk as file /backup/DB2/1_3511_888709478.dbf
archived log for thread 1 with sequence 3512 is already on disk as file /backup/DB2/1_3512_888709478.dbf
archived log for thread 1 with sequence 3513 is already on disk as file /backup/DB2/1_3513_888709478.dbf
archived log file name=/backup/DB2/1_3511_888709478.dbf thread=1 sequence=3511
archived log file name=/backup/DB2/1_3512_888709478.dbf thread=1 sequence=3512
archived log file name=/backup/DB2/1_3513_888709478.dbf thread=1 sequence=3513
media recovery complete, elapsed time: 00:00:02
Finished recover at 04-FEB-16
Finished Duplicate Db at 04-FEB-16
released channel: d1
released channel: d2
released channel: stby

RMAN&gt; **end-of-file**</pre>

<br>Check to see if everything is working. Switch your logfile on the primary to see if you have the same sequence# on both sides.</br>

Primary (node1).

<pre>SQL&gt; alter system switch logfile;

System altered.

SQL&gt; select max(sequence#) from v$log_history;

MAX(SEQUENCE#)
--------------
3517</pre>

<br>Standby (node2).</br>

<pre>SQL&gt; select max(sequence#) from v$log_history;

MAX(SEQUENCE#)
--------------
3517</pre>

<br>I needed to force a re-read of my tnsnames.ora on the primary. You can do this by changing log_archive_dest_state to 'defer' and then back to 'enable'  on the primary.</br>

<pre>alter system set log_archive_dest_state_2=defer scope=both;
alter system set log_archive_dest_state_2=enable scope=both;</pre>

<br>Finally, check your database role on the standby (node2).</br>

<pre>SQL&gt; select database_role from v$database;

DATABASE_ROLE
----------------
PHYSICAL STANDBY</pre>