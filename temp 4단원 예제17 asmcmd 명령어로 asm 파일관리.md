<pre>
<br>

※ 밑에 실습들은 11g ASM에서 함
  7. 백업과 복구는 19c ASM에서 할 예쩡


  
● ASM
  - 오라클 데이터베이스용 스토리지 관리 기능 
  - 데이터 파일·로그 파일 등을 디스크 그룹으로 묶어 자동으로 배치/미러링/리밸런싱 하여 성능과 가용성 향상


<img width="885" height="632" alt="Rade 0" src="https://github.com/user-attachments/assets/22056915-dbf4-4a72-a87f-90a425432f09" />

  
<img width="811" height="632" alt="Raid 1" src="https://github.com/user-attachments/assets/4015c818-9bad-49f0-9ae5-6684219ae3cf" />


  RAID 0 + 1 구성
   장점 : 스트라이핑 + 미러링 동시 사용  →  성능↑ + 백업
   단점 : 비쌈






<img width="590" height="255" alt="raid5" src="https://github.com/user-attachments/assets/b624084a-ebc1-43f8-baf4-84cfa9c21080" />

  RAID 5 : 디스크로 스트라이핑 미러링 둘 다 함
   장점 : 저렴함
   단점 : 너무 느림 ( 자주 쓰기가 일어나는 파일은 RAID 5로 구성 X (Redo Log File ...) )




  ASM 의 장점
  - Oracle이 알아서 하드웨어의 RAID 기법 적용 ( 미러링 + 스트라이핑 알아서 셋팅 )
  - 운영 중에 디스크 추가 가능
  - 데이터 이행 시 디스크만 빼서 새 서버에 붙이면 끝 ( PUMP 작업 필요 없음 )



  

- 파일 확인

select file_name from dba_data_files;

FILE_NAME
--------------------------------------------------------------------------------
+DATA/orcl/datafile/users.259.796857625
+DATA/orcl/datafile/undotbs1.258.796857625
+DATA/orcl/datafile/sysaux.257.796857623
+DATA/orcl/datafile/system.256.796857621
+DATA/orcl/datafile/example.265.796857803
+DATA/orcl/datafile/ts02.267.1226325885
+DATA/orcl/datafile/ts01.268.1228400881
+DATA/orcl/datafile/ts07.269.1228401057

  
select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
+DATA/orcl/controlfile/current.260.796857737
+FRA/orcl/controlfile/current.256.796857739

  
select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
+DATA/orcl/onlinelog/group_3.263.796857759
+FRA/orcl/onlinelog/group_3.259.796857763
+DATA/orcl/onlinelog/group_2.262.796857753
+FRA/orcl/onlinelog/group_2.258.796857757
+DATA/orcl/onlinelog/group_1.261.796857743
+FRA/orcl/onlinelog/group_1.257.796857749

6 rows selected.


- bash에서 파일 찾아가기

[orcl:~]$ . oraenv
ORACLE_SID = [orcl] ? +ASM
The Oracle base for ORACLE_HOME=/u01/app/oracle/product/11.2.0/grid is /u01/app/oracle
[+ASM:~]$
[+ASM:~]$ asmcd
-bash: asmcd: command not found
[+ASM:~]$ asmcmd
ASMCMD>
ASMCMD> cd data/orcl/datafile
ASMCMD> ls
EXAMPLE.265.796857803
SYSAUX.257.796857623
SYSTEM.256.796857621
TS01.268.1228400881
TS02.267.1226325885
TS07.269.1228401057
UNDOTBS1.258.796857625
USERS.259.796857625


  
- ASM은 반드시 RMAN으로 백업

SQL> archive log list
SQL> shutdown  immediate
SQL> startup mount
SQL> alter database archivelog;
SQL> alter database open;

[orcl:~]$ rman target / nocatalog

RMAN> configure controlfile autobackup on;
RMAN> backup database;



  
- rm으로 datafile 삭제  

ASMCMD> rm TS01.268.1228400881  (shutdown 시 실행가능)

ORA-15032: not all alterations performed
ORA-15028: ASM file '+data/orcl/datafile/TS01.268.1228400881' not dropped; currently being accessed (DBD ERROR: OCIStmtExecute)


- 삭제 datafile 파일 복원

startup mount;
select * from v$recover_file; -- 파일번호가 뜸 ( ex) 7 )
rman target /
RMAN> restore datafile 7;
RMAN> recover datafile 7;

경로를 알면 이렇게 해도됨
RMAN> restore datafile '+data/orcl/datafile/TS01.268.1228400881';
RMAN> recover datafile '+data/orcl/datafile/TS01.268.1228400881';

SQL> alter database open;




● ASM 디스크 그룹 : 여러 개의 물리적 디스크를 하나의 논리적 단위로 묶어서 관리하는 스토리지 풀

  
- 디스크 그룹 조회
  
select name, total_mb, free_mb from v$asm_diskgroup;
NAME                             TOTAL_MB    FREE_MB
------------------------------ ---------- ----------
DATA                                 9216       5260
FRA                                  9216       6828


select group_number, mount_status, path, total_mb from v$asm_disk;
GROUP_NUMBER MOUNT_S PATH                   TOTAL_MB
------------ ------- -------------------- ----------
           1 CACHED  ORCL:ASMDISK01             2304
           1 CACHED  ORCL:ASMDISK02             2304
           1 CACHED  ORCL:ASMDISK03             2304
           1 CACHED  ORCL:ASMDISK04             2304
           2 CACHED  ORCL:ASMDISK05             2304
           2 CACHED  ORCL:ASMDISK06             2304
           2 CACHED  ORCL:ASMDISK07             2304
           2 CACHED  ORCL:ASMDISK08             2304
           0 CLOSED  ORCL:ASMDISK13                0   -- 1 CACHED  ORCL:ASMDISK13 2304 (밑에 명령 실행 시)
           0 CLOSED  ORCL:ASMDISK12                0
           0 CLOSED  ORCL:ASMDISK11                0
           0 CLOSED  ORCL:ASMDISK10                0
           0 CLOSED  ORCL:ASMDISK09                0


- 디스크 추가
[+ASM:~]$ sqlplus / as sysasm
alter diskgroup data add disk 'ORCL:ASMDISK13' rebalance power 2;

  
- FRA 디스크 그룹에 디스크 추가
alter diskgroup fra add disk 'ORCL:ASMDISK12' rebalance power 2;


- 디스크 삭제
alter diskgroup data drop disk asmdisk13 rebalance power 2;





● ASM 디스크 인스턴스 

  - 디스크 그룹(ASM 디스크)을 인식하고, 데이터베이스 인스턴스에 저장공간을 제공
  - 파일을 직접 다루지 않음, 데이터 파일·리두 로그·컨트롤 파일 등의 배치를 자동으로 관리
  - 디스크 추가/제거 시 데이터를 재분산(Rebalance)하여 성능과 가용성 유지
  - 스토리지 관련 메타데이터와 백그라운드 프로세스를 통해 ASM 환경을 운영

[+ASM:~]$ sqlplus / as sysasm
show parameter spfile  -- orcl (ORACLE) Parameter 파일과 다르게 나옴

show parameter asm     -- limit : 1
  
alter system set asm_power_limit=7 scope=both;
  
DB Instacne Shutdown  →  ASM Instance Shutdown  →   ASM Instance Startup  →  DB Instance Startup
  



  
</pre>
