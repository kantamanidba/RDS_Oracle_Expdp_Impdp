# RDS_Oracle_Expdp_Impdp

Step by step procedure for schema backup and restore accross the environments@AWS Oracle RDS.
 
Below is the procedure for taking expdp backup at source(Usually at production):

```
declare
--L_DP_HANDLE varchar2(10000);
L_DP_HANDLE number;
begin
L_DP_HANDLE := DBMS_DATAPUMP.open(OPERATION => 'EXPORT',JOB_MODE => 'SCHEMA',version=> 'LATEST',JOB_NAME=>'SCHEMA_NAME_EXPORT');

DBMS_DATAPUMP.ADD_FILE(HANDLE    => L_DP_HANDLE,
                       filename  => 'dump_file_name.dmp',
		       filetype => dbms_datapump.ku$_file_type_dump_file,
                       directory => 'DATA_PUMP_DIR',
                       REUSEFILE => 1);
                    
DBMS_DATAPUMP.ADD_FILE(HANDLE    => L_DP_HANDLE,
                       filename  => 'log_file_name.log',
                       directory => 'DATA_PUMP_DIR',
                       FILETYPE  => DBMS_DATAPUMP.KU$_FILE_TYPE_LOG_FILE,
                       REUSEFILE => 1);
dbms_datapump.metadata_filter(handle => l_dp_handle, name => 'SCHEMA_LIST', value => '''SCHEMA_NAME''');                    
Dbms_DataPump.Start_Job(l_dp_handle);
DBMS_DATAPUMP.detach(l_dp_handle);
EXCEPTION
WHEN OTHERS THEN
dbms_output.put_line(sqlerrm);
END;
```
command for listing the contents of DATA_PUMP_DIR directory:

select * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('DATA_PUMP_DIR')) where trunc(mtime)=trunc(sysdate) order by mtime;

command for view the contents of log file:

select * from table(RDSADMIN.RDS_FILE_UTIL.READ_TEXT_FILE('DATA_PUMP_DIR','log_file_name.log'));


Procedure for moving dumpfile to S3 bucket:
```
SELECT rdsadmin.rdsadmin_s3_tasks.upload_to_s3(
      p_bucket_name    =>  'BUCKET_NAME',
      p_prefix         =>  'dump_file_name.dmp',
      p_s3_prefix      =>  '',
      p_directory_name =>  'DATA_PUMP_DIR')
   AS TASK_ID FROM DUAL;
```
At Target server:

command for listing the contents of DATA_PUMP_DIR directory:

select * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('DATA_PUMP_DIR')) where trunc(mtime)=trunc(sysdate) order by mtime;


Procedure for dowloading the dumpfile from S3 bucket:
```
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
      p_bucket_name    =>  'BUCKET_NAME', 
	p_s3_prefix         =>  'dump_file_name.dmp',      
      p_directory_name =>  'DATA_PUMP_DIR') 
   AS TASK_ID FROM DUAL; 
```
command for listing the contents of DATA_PUMP_DIR directory:

select * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('DATA_PUMP_DIR')) where trunc(mtime)=trunc(sysdate) order by mtime;



Below is the procedure for restoring the dumpfile backup to target schema (Usually at Dev/Test):

```
declare
--L_DP_HANDLE varchar2(10000);
L_DP_HANDLE number;
begin
L_DP_HANDLE := DBMS_DATAPUMP.open(OPERATION => 'IMPORT',JOB_MODE => 'SCHEMA',version=> 'LATEST',JOB_NAME=>'SCHEMA_NAME_IMPORT');

DBMS_DATAPUMP.ADD_FILE(HANDLE    => L_DP_HANDLE,
			filename  => 'dump_file_name.dmp',
			filetype => dbms_datapump.ku$_file_type_dump_file,
			directory => 'DATA_PUMP_DIR'
			);

DBMS_DATAPUMP.ADD_FILE(HANDLE    => L_DP_HANDLE,
			filename  => 'import_schema_name.log',
			directory => 'DATA_PUMP_DIR',
			FILETYPE  => DBMS_DATAPUMP.KU$_FILE_TYPE_LOG_FILE
			);

DBMS_DATAPUMP.METADATA_REMAP(L_DP_HANDLE,'REMAP_SCHEMA','source_schema_name','target_schema_name');
Dbms_DataPump.Start_Job(l_dp_handle);
DBMS_DATAPUMP.detach(l_dp_handle);
EXCEPTION
WHEN OTHERS THEN
dbms_output.put_line(sqlerrm);
END;
/
```

command for listing the contents of DATA_PUMP_DIR directory:

select * from table(RDSADMIN.RDS_FILE_UTIL.LISTDIR('DATA_PUMP_DIR')) where trunc(mtime)=trunc(sysdate) order by mtime;

command for view the contents of log file:

select * from table(RDSADMIN.RDS_FILE_UTIL.READ_TEXT_FILE('DATA_PUMP_DIR','import_schema_name.log'));
