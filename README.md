![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Create [ZTrack] Audit Database Base Tables And Job With SQL
**Post Date: November 20, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>The following SQL logic will create the base tables, and first Job for the ZTRACK system.</p>      


## SQL-Logic
```SQL
use ZTRACK;
set nocount on
 
 
insert into ABOUT_ZTRACK 
    (
        zt_object_type
    ,   zt_object_name
    --, zt_object_logic
    ,   zt_description
    )
values
    (
        'TABLE'
    ,   'ZT_TABLE_GET_DATABASE_SIZE_INFO'
    --, 'insert my query logic here if needed'
    ,   'This table holds database size information.  Basically this is copy of the sys.master_files table.  
        Its here so we can gather historical information such as growth etc.  The table is updated by a procedure [ZT_USP_GET_DATABASE_SIZE_INFO],
        and this is run from a job called ZTRACK GET INFO - DATABASE SIZE.'
    )
 
IF OBJECT_ID('ZTRACK..ZT_TABLE_GET_DATABASE_SIZE_INFO') IS NOT NULL
    DROP TABLE ZT_TABLE_GET_DATABASE_SIZE_INFO
CREATE TABLE ZT_TABLE_GET_DATABASE_SIZE_INFO
(
    [database_id]           INT
,   [file_id]               INT
,   [file_type_desc]        NVARCHAR(255)
,   [name]                  NVARCHAR(255)
,   [physical_name]         NVARCHAR(4000)
,   [state_desc]            NVARCHAR(255)
,   [size]                  INT
,   [max_size]              INT
,   [growth]                INT
,   [is_sparse]             BIT
,   [is_percent_growth]     BIT
,   [date_captured]         DATETIME
)
go
 
insert into ABOUT_ZTRACK 
    (
        zt_object_type
    ,   zt_object_name
    --, zt_object_logic
    ,   zt_description
    )
values
    (
        'STORED PROCEDURE'
    ,   'ZT_USP_GET_DATABASE_SIZE_INFO'
    --, 'insert my query logic here if needed'
    ,   'This procedure drives the update process for getting database size information into the ZT table ZT_TABLE_GET_DATABASE_SIZE_INFO.'
    );
go
 
IF OBJECT_ID('ZTRACK..ZT_USP_GET_DATABASE_SIZE_INFO') IS NOT NULL
    DROP PROCEDURE ZT_USP_GET_DATABASE_SIZE_INFO
GO
 
CREATE PROC ZT_USP_GET_DATABASE_SIZE_INFO
AS
BEGIN
SET NOCOUNT ON
INSERT INTO ZT_TABLE_GET_DATABASE_SIZE_INFO
(
    [database_id]
,   [file_id]
,   [file_type_desc]
,   [name]
,   [physical_name]
,   [state_desc]
,   [size]
,   [max_size]
,   [growth]
,   [is_sparse]
,   [is_percent_growth]
,   [date_captured]
)
SELECT
    [database_id]
,   [file_id]
,   [type_desc]
,   [name]
,   [physical_name]
,   [state_desc]
,   [size]
,   [max_size]
,   [growth]
,   [is_sparse]
,   [is_percent_growth]
,   GETDATE()
FROM sys.master_files
SET NOCOUNT OFF
END
GO
 
use ZTRACK;
set nocount on
 
insert into ABOUT_ZTRACK 
(
    zt_object_type
,   zt_object_name
--, zt_object_logic
,   zt_description
)
values
(
    'AGENT JOB'
,   'ZTRACK GET INFO - DATABASE SIZE'
--, 'insert my query logic here if needed'
,   'This job runs a stored procedure in the ZTRACK Database to collect and store database file information.'
)
 
 
USE [msdb]
GO
 
IF EXISTS(SELECT NAME FROM MSDB..SYSJOBS WHERE NAME = 'ZTRACK GET INFO - DATABASE SIZE')
    EXEC MSDB..SP_DELETE_JOB @JOB_NAME = 'ZTRACK GET INFO - DATABASE SIZE'
 
BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0
 
IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
END
DECLARE @jobId BINARY(16)
EXEC @ReturnCode =  msdb.dbo.sp_add_job @job_name=N'ZTRACK GET INFO - DATABASE SIZE',
@enabled=1,
@notify_level_eventlog=0,
@notify_level_email=0,
@notify_level_netsend=0,
@notify_level_page=0,
@delete_level=0,
@description=N'This job runs a stored procedure in the ZTRACK Database to collect and store database file information.',
@category_name=N'[Uncategorized (Local)]',
@owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
 
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Get Database Size Info',
@step_id=1,
@cmdexec_success_code=0,
@on_success_action=1,
@on_success_step_id=0,
@on_fail_action=2,
@on_fail_step_id=0,
@retry_attempts=0,
@retry_interval=0,
@os_run_priority=0, @subsystem=N'TSQL',
@command=N'EXEC ZTRACK..ZT_USP_GET_DATABASE_SIZE_INFO',
@database_name=N'ZTRACK',
@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId, @name=N'ZTRACK GET INFO - DATABASE SIZE',
@enabled=1,
@freq_type=4,
@freq_interval=1,
@freq_subday_type=1,
@freq_subday_interval=0,
@freq_relative_interval=0,
@freq_recurrence_factor=0,
@active_start_date=20130213,
@active_end_date=99991231,
@active_start_time=0,
@active_end_time=235959
 
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:
  
GO
 
use ZTRACK;
set nocount on
 
insert into ABOUT_ZTRACK 
(
    zt_object_type
,   zt_object_name
,   zt_object_logic
,   zt_description
)
values
(
    'QUERY'
,   'GET DATABASE SIZE OVER TIME'
,   'SELECT
    DB_NAME(database_id) AS DatabaseName,
    collect_date AS CollectionDate,
    ((SUM(size))*8)/1024/1024 AS SizeInGB
    FROM ZT_TABLE_GET_DATABASE_SIZE_INFO
    GROUP BY database_id,collect_date
    ORDER BY DatabaseName,collect_date'
,   'This query gives you the data size info for the databases over time'
);
go
 
insert into ABOUT_ZTRACK 
(
    zt_object_type
,   zt_object_name
,   zt_object_logic
,   zt_description
)
values
(
'QUERY'
,   'GET LOG SIZE OVER TIME'
,   'SELECT
    SELECT DB_NAME(database_id) AS DatabaseName,
    (CASE file_type_desc
    WHEN ''ROWS''THEN ''Data''
    WHEN ''LOG'' THEN ''Log''
    END) AS FileType,
    physical_name AS PhysicalPath,
    collect_date AS CollectionDate,
    ((SUM(size))*8)/1024/1024 AS FileSizeInGB
    FROM ZT_TABLE_GET_DATABASE_SIZE_INFO
    GROUP BY database_id,physical_name,file_type_desc,collect_date
    ORDER BY DatabaseName,collect_date'
,   'This query gives you the log size info for the databases over time'
);
go
 
WAITFOR DELAY '00:00:05'
GO
 
EXEC MSDB..SP_START_JOB @JOB_NAME = 'ZTRACK GET INFO - DATABASE SIZE'
 
USE ZTRACK;
SELECT * FROM ABOUT_ZTRACK

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

