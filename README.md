oracle statepack report


目錄


1.目的與基本檢查	3

2.Statspack安裝步驟	4
 
3.參考文獻	10


























1.目的與基本檢查

	目的:為了能夠每天觀察DB運作是否正常，有無存在小問題或不良SQL影響效能的狀況，如果久沒注意，長時間運作下來可能會產生極大障礙，造成旁大客訴。
                
             





	問題描述：當維運人員很長一段時間沒注意DB上的效能問題，以及沒產出相關report檢查DB效能以及resource使用狀況，在經過一段時間後，許多日積月累的小問題就造成了DB crash的原因。
                     

                         
	解決方式：需建立一份產DB report 的步驟，每天定時產出report，並觀察有無效能問題存在，避免小問題引發大障礙。


	請先抓取此SOP所需的檢查檔案(連結如下，已建立超連結，壓ctrl可點選)
\\tccfs01\PID\Public\安裝工具\Sendmail




2.Statspack安裝步驟
注意：若已有安裝過Statpack此步驟可忽略

	步驟1：安裝Statspack準備動作(如已安裝此步驟不用執行)

[oracle@AAA-11GStaging ~]$ sqlplus /nolog

SQL*Plus: Release 11.2.0.3.0 Production on Fri Aug 7 10:43:46 2015

Copyright (c) 1982, 2011, Oracle.  All rights reserved.

SQL> connect / as sysdba(注意 需以DBA權限登入)
Connected.
SQL> show parameter timed_statistics;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
timed_statistics                     boolean      TRUE(檢查是否為TRUE)

SQL> show parameter job_queue_processes ;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
job_queue_processes                  integer     1000(檢查是否大於0)

	為了能夠建立自動任務，執行數據收集，此參數必須大於0(job_queue)
	使收集的時間信息存在V$sessstats和V$sysstats等動態性能視圖中需將timed_statistics值設定為TRUE


	步驟2：建立Tablespace for statspack使用 


SQL> connect / as sysdba(注意 需以DBA權限登入)
Connected.
SQL> create tablespace perfstat datafile '/home/oracle/app/oracle/oradata/orcl/PERFSTAT.dbf' size 1024m;
 (此路徑請依自己主機環境更改)
Tablespace created.

	增加一tablespace給statspack使用，安裝最低需求為100m可視情況調整，statspack的owner為PERFSTAT




	步驟3：執行安裝Statspack之腳本 

SQL> connect / as sysdba(注意 需以DBA權限登入)
Connected.
SQL>@$ORACLE_HOME/rdbms/admin/spcreate.sql (腳本路徑位置)
Choose the PERFSTAT user's password
-----------------------------------


	腳本會創建用戶perfstat，需要指定此用戶密碼(務必使用強度為長度至少八碼英數混合的密碼)。
輸入 perfstat_password 的值: 
	需要輸入用戶perfstat使用的表空間：指定步驟2新建的表空間即可。
輸入 default_tablespace 的值:   perfstat
	需要指定用戶perfstat使用的臨時表空間。
輸入 temporary_tablespace 的值: temp
	如果出現錯誤，可以運行腳本刪除相關內容：@/oracle_home/rdbms/admin/spdrop.sql(注意：也要在sysdba下運行腳本刪除相關對象)然後再重新運行腳本安裝。


	步驟4：產生系統快照 (snapshop)

(如果你剛執行完@spcreate，則oracle默認將當前用戶切換为perfstat 。)
SQL>show user
USER is "PERFSTAT"

SQL> execute statspack.snap;
PL/SQL procedure successfully completed.

SQL> execute statspack.snap;
PL/SQL procedure successfully completed.
(需執行二次快照，才可產出report資料)

	運行statspack.snap可以產生系統快照，運行兩次，產生兩次快照









	步驟5：執行report程式，產生report檔

SQL> connect perfstat/"XXXXXXXX" (注意 需以PERFSTAT登入)
Connected.
SQL>@$ORACLE_HOME/rdbms/admin/spreport.sql (腳本路徑位置)
Instance     DB Name        Snap Id   Snap Started    Level Comment
------------ ------------ --------- ----------------- ----- --------------------
T3VASMD      T3VASMD           1 10 Aug 2015 12:49     5
                                  2 10 Aug 2015 12:49     5
                                  3 10 Aug 2015 14:11     5



Specify the Begin and End Snapshot Ids
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Enter value for begin_snap  (輸入開始的snap ID，可從列出的清單選擇)
Enter value for end_snap: (輸入結束的snap ID，可從列出的清單選擇)
Enter value for report_name) (產出的report路徑，可自行設定)
	可由snap id選擇所要產出報表的時間點，在將報表產出在設定的路徑上。
	一開始的Level comment 預設為5，此為產出report資料的詳細度，正常狀況下5是足夠的。

	步驟6：設定自動產生snapshot檔之shellscript

[oracle@staging~]$ vim perfmon.sh
#!/bin/bash

PATH=$PATH:$HOME/bin

export PATH
unset USERNAME

ORACLE_SID=依自行環境設定; export ORACLE_SID

#ORACLE_BASE=/oracle/app/oracle; export ORACLE_BASE
ORACLE_BASE=/oracle/app/oracle; export ORACLE_BASE
ORACLE_HOME=/oracle/app/oracle/product/11.2.0.3; export ORACLE_HOME
PATH=$ORACLE_HOME/bin:$PATH; export PATH

LD_LIBRARY_PATH=$ORACLE_HOME/lib; export LD_LIBRARY_PATH

NLS_LANG=AMERICAN_AMERICA.ZHT16BIG5; export NLS_LANG

	(注意:黃字體部份請依主機環境設定)

sqlplus /nolog <<EOF
connect / as sysdba
execute perfstat.statspack.snap;
exit
EOF
	自動執行建立snapshop，一些預設值需依各主機環境變更

	步驟7：設定自動清除大於30天的snapshop log

[oracle@staging~]$ vim purgeperf.sh
#!/bin/bash

PATH=$PATH:$HOME/bin

export PATH
unset USERNAME


ORACLE_SID=依自行環境設定; export ORACLE_SID

#ORACLE_BASE=/oracle/app/oracle; export ORACLE_BASE
ORACLE_BASE=/oracle/app/oracle; export ORACLE_BASE
ORACLE_HOME=/oracle/app/oracle/product/11.2.0.3; export ORACLE_HOME
PATH=$ORACLE_HOME/bin:$PATH; export PATH

LD_LIBRARY_PATH=$ORACLE_HOME/lib; export LD_LIBRARY_PATH

NLS_LANG=AMERICAN_AMERICA.ZHT16BIG5; export NLS_LANG
	(注意:黃字體部份請依主機環境設定)

sqlplus /nolog <<EOF
connect / as sysdba
delete from perfstat.stats\$snapshot where snap_time < sysdate -30;
commit;
exit
EOF
	自動清除大於30天之snapshop log，一些預設值需依各主機環境變更

	步驟8：建立Crontab 自動執行此二隻shellscript 

0 * * * *  sh /app/oracle/bin/perfmon.sh
5 * * * * sh /app/oracle/bin/purgeperf.sh
(新增此二個Shellscript crontab)


	將此二隻shellscript加入crontab中，即可自動產生snapshop檔以及自動清除30天以前的snapshop log檔。









	步驟9：建立Crontab 自動產出昨日報表 

[oracle@staging~]$ vim autostatsreport.sh

#!/bin/bash

. ~/.bash_profile

sqlplus -S  <<EOF
conn / as sysdba
column minsnapid noprint new_value begsnapid
column maxsnapid noprint new_value endsnapid
column spreportname noprint new_value reportname

select min(snap_id) minsnapid  ,  min(snap_id) + 24 maxsnapid
from perfstat.stats\$snapshot
where SNAP_TIME > trunc(sysdate) -1  ;


select 'sp_'||to_char(sysdate -1 , 'yyyy_mm_dd') ||'.lst' spreportname
from dual;

define begin_snap=&begsnapid
define end_snap=&endsnapid
define report_name=/tmp/&reportname

@$ORACLE_HOME/rdbms/admin/spreport.sql
EOF 
(新增此shell並加入crontab)
* 8 * * * sh /app/oracle/bin/ autostatsreport.sh

	將此隻shellscript加入crontab中，即可自動產生昨日的statspack report報表至/tmp底下。

步驟9：自動發送報表至mail
1.	需開通server至mail server firewall
2.	開通後即可將步驟9產出至report設定crontab自動發送至指定Email帳戶
3.	請將page3所抓取的附件(sendEmail.pl)上傳到/root/script
4.	將此shell加入crontab

[oracle@staging~]$ vim SendDBreport.sh
#!/bin/bash

DAY=`date -d "1 day ago" +%Y_%m_%d`

reportfile="/tmp/sp_${DAY}.lst"
reportday="`date -d "1 day ago" +%Y_%m_%d`"
/root/script/sendEmail.pl -f SDMPDB@taiwanmobile.com -t austinwu@taiwanmobile.com  easonhsieh@taiwanmobile.com –cc jason1wang@taiwanmobile.com -s 172.20.2.131:25 -u "[TOP SQL]_SDMPDB" -a $reportfile -m "[TOP SQL]_SDMPDB$reportday” statsreport
(紅字部份請依照自己的服務名稱修改)
(空格可輸入多筆收件者Email帳號)
(新增此shell並加入crontab)

5.	即可自動收到mail
 



























參考文獻
	Oracle security Guide
	Oracle Study Guide
