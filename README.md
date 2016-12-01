OGG 12.2经典模式安装步骤
1.	使用oracle用户，上传介质到服务器并解压，得到文件夹fbo_ggs_Linux_x64_shiphome
[oracle@host2 oraadm]:oggtgt> unzip fbo_ggs_Linux_x64_shiphome.zip

2.	打开vnc，进入图形界面
[oracle@host2 oraadm]:oggtgt> vncserver 

New 'X' desktop is host2:2

Starting applications specified in /home/oracle/.vnc/xstartup
Log file is /home/oracle/.vnc/host2:2.log

 

3.	弹出的命令行窗口中执行以下操作：
[oracle@host2 oraadm]:oggtgt> cd fbo_ggs_Linux_x64_shiphome/Disk1/
[oracle@host2 Disk1]:oggtgt> ./runInstaller
 

4.	选项会自动填入，可以自行更改软件安装位置，database location不建议更改，连续点击下一步即可安装：
 

5.	退出vnc，回到命令行页面。查询之前的安装目录，查看安装组件，并登陆ggsci
[oracle@host2 Disk1]:oggtgt> cd /opt/goldengate/
[oracle@host2 goldengate]:oggtgt> ls
[oracle@host2 goldengate]:oggtgt> ./ggsci
--作者：john5480
--地址：http://blog.itpub.net/26110799/
Oracle GoldenGate Command Interpreter for Oracle
Version 12.2.0.1.1 OGGCORE_12.2.0.1.0_PLATFORMS_151211.1401_FBO
Linux, x64, 64bit (optimized), Oracle 11g on Dec 12 2015 00:54:38
Operating system character set identified as UTF-8.

Copyright (C) 1995, 2015, Oracle and/or its affiliates. All rights reserved.



GGSCI (host2) 1>

6.	进入数据库，打开supplementary log
select force_logging,supplemental_log_data_min from v$database;
alter database force logging;
alter database add supplemental log data;
SQL> select force_logging,supplemental_log_data_min from v$database;

FOR SUPPLEME
--- --------
YES YES

7.	创建表空间，建用户
CREATE TABLESPACE DBADATATBS DATAFILE
'/******/dbadatatbs01.dbf' SIZE 2048M AUTOEXTEND OFF
LOGGING
ONLINE
EXTENT MANAGEMENT LOCAL AUTOALLOCATE
BLOCKSIZE 8K
SEGMENT SPACE MANAGEMENT AUTO
FLASHBACK ON;

--（如果用户已存在，则变更权限即可）
CREATE USER GGSYNC
IDENTIFIED BY ggsync
DEFAULT TABLESPACE DBADATATBS
TEMPORARY TABLESPACE TEMP
PROFILE DEFAULT
ACCOUNT UNLOCK;
-- 2 Roles for GGSYNC
GRANT CONNECT TO GGSYNC;
GRANT RESOURCE TO GGSYNC;
ALTER USER GGSYNC DEFAULT ROLE ALL;
-- 11 System Privileges for GGSYNC
GRANT ALTER SESSION TO GGSYNC;
GRANT CREATE SESSION TO GGSYNC;
GRANT SELECT ANY TABLE TO GGSYNC;
GRANT SELECT ANY DICTIONARY TO GGSYNC;
GRANT CREATE TABLE TO GGSYNC;
GRANT UNLIMITED TABLESPACE TO GGSYNC;
GRANT FLASHBACK ANY TABLE TO GGSYNC;
GRANT INSERT ANY TABLE TO GGSYNC;
GRANT DELETE ANY TABLE TO GGSYNC;
GRANT UPDATE ANY TABLE TO GGSYNC;
GRANT ALTER ANY TABLE TO GGSYNC;
grant create view to ggsync;
grant create job to ggsync;
grant select any dictionary to ggsync;
grant dba to ggsync;
grant execute on dbms_capture_adm to ggsync;
grant SELECT on dba_clusters to ggsync;
grant SELECT on V_$DATABASE to ggsync;
grant select on sys.logmnr_buildlog to ggsync;
grant EXECUTE on DBMS_FLASHBACK to ggsync;
grant execute on DBMS_CAPTURE_ADM to ggsync;
grant execute on DBMS_STREAMS to ggsync;  
grant EXECUTE_CATALOG_ROLE to ggsync;

8.	增加心跳表，创建GLOBALS文件
GGSCI (host1 as ggsync@oggsrc) 21> edit param ./GLOBALS
--输入以下两行内容
CHECKPOINTTABLE ggsync.gg_checkpoint_tab
GGSCHEMA ggsync
GGSCI (host2) 8> dblogin userid ggsync,password ggsync
GGSCI (host2 as ggsync@oggtgt) 10> add HEARTBEATTABLE
GGSCI (host2 as ggsync@oggtgt) 11> ADD CHECKPOINTTABLE ggsync.gg_checkpoint_tab  

--作者：john5480
--地址：http://blog.itpub.net/26110799/

9. 创建 MGR参数文件并启动 MGR进程
./ggsci
edit param mgr
--输入如下内容:
port 7809
DYNAMICPORTLIST 7810-7909
PURGEOLDEXTRACTS ./dirdat/*/*,usecheckpoints,minkeepdays 3
autostart er *
autorestart er *,retries 5,waitminutes 7,resetminutes 60
lagreporthours 1
laginfominutes 5
lagcriticalminutes 5                  

GGSCI (host1) 5> stop mgr
Manager process is required by other GGS processes.
Are you sure you want to stop it (y/n)?y

Sending STOP request to MANAGER ...
Request processed.
Manager stopped.


GGSCI (host1) 6> start mgr
Manager started.GGSCI (oggtest) 3> info all
Program Status Group Lag at Chkpt Time Since Chkpt
MANAGER RUNNING

9.	创建抽取进程
创建目录
[oracle@host1 goldengate]:oggsrc> mkdir dirdat/e_oggsrc
GGSCI (host1) 7> add extract e_oggsrc, tranlog, begin now
GGSCI (host1) 8> add exttrail ./dirdat/e_oggsrc/ea, extract e_oggsrc, megabytes 500
GGSCI (host1) 9> edit param e_oggsrc
EXTRACT e_oggsrc

SETENV (ORACLE_HOME = /app/oracle/ora11g)
SETENV (ORACLE_SID=oggsrc)
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
userid ggsync, password ggsync

EXTTRAIL ./dirdat/e_oggsrc/ea
DISCARDFILE ./dirrpt/e_oggsrc.dsc,append,megabytes 2000

--EXCLUDE table
TABLEEXCLUDE *.DBMS_TAB
-- gg sync
table scott.*;

10.	配置传输进程
首先在目标库位置创建接受目录
[oracle@host2 goldengate]:oggtgt> mkdir dirdat/r_oggsrc
GGSCI (host1 as ggsync@oggsrc) 19> add extract t_oggtgt,exttrailsource ./dirdat/e_oggsrc/ea,begin now
GGSCI (host1 as ggsync@oggsrc) 20> add rmttrail ./dirdat/r_oggsrc/ra,extract t_oggtgt,megabytes 500

GGSCI (host1 as ggsync@oggsrc) 21> edit param t_oggtgt
EXTRACT t_oggtgt
SETENV (ORACLE_HOME = /app/oracle/ora11g)
SETENV (ORACLE_SID=oggsrc)
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
userid ggsync, password ggsync
passthru
RMTHOST 10.120.89.205, MGRPORT 7809, compress
RMTTRAIL ./dirdat/r_oggsrc/ra
DISCARDFILE ./dirrpt/t_oggtgt.dsc,append,megabytes 2000
DISCARDROLLOVER AT 05:30 ON Friday
REPORTROLLOVER AT 05:30 ON Friday
REPORTCOUNT EVERY 10 MINUTES, RATE
-- gg sync
table scott.*;

11.	配置目标复制进程（目标库）
GGSCI (host2 as ggsync@oggtgt) 5> edit param r_oggsrc
replicat r_oggsrc

SETENV (ORACLE_HOME = /opt/oracle/product/11gR2/db)
SETENV (ORACLE_SID=orcltest)
SETENV (NLS_LANG="AMERICAN_AMERICA.ZHS16GBK")
userid ggsync, password ggsync

--HANDLECOLLISIONS
REPERROR DEFAULT,ABEND
DISCARDFILE ./dirrpt/r_oggsrc.dsc,append,megabytes 2000
DISCARDROLLOVER AT 05:30 ON Friday
REPORTROLLOVER AT 05:30 ON Friday
REPORTCOUNT EVERY 10 MINUTES, RATE
numfiles 5000
CHECKPOINTSECS 30
GROUPTRANSOPS 10000
MAXTRANSOPS 30000
ASSUMETARGETDEFS
dynamicresolution
WILDCARDRESOLVE DYNAMIC
ALLOWDUPTARGETMAP

map scott.*,target scott.*;

12.	初始化数据
1.初始化所有表数据
从源库复制需要同步的表至目标库
2.启用参数 HANDLECOLLISIONS
去掉 HANDLECOLLISIONS 前的，”--”
（如果是重新同步个别表，则在 map 后面添加 HANDLECOLLISIONS，
例如：map test1.t1, target test.t1 HANDLECOLLISIONS;)
3.启动恢复进程 r_testdb
4.等待恢复进程同步完成之后，停止恢复进程 r_testdb
5.禁用 HANDLECOLLISIONS
6.启动恢复进程

