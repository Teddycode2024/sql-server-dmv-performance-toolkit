# sql-server-dmv-performance-toolkit
A performance-focused toolkit of SQL Server DMV scripts used to analyze CPU usage, memory pressure, disk latency, blocking chains, and wait statistics. This collection reflects scripts I’ve gathered through real-world experience and regularly apply when diagnosing and resolving production performance issues.

Top 50 Essential Daily Health Checks for SQL Server DBA


1. Script to Identify expensive query by Resource 

---------
SELECT TOP(50) qs.execution_count AS [Execution Count],
(qs.total_logical_reads)*8/1024.0 AS [Total Logical Reads (MB)],
(qs.total_logical_reads/qs.execution_count)*8/1024.0 AS [Avg Logical Reads (MB)],
(qs.total_worker_time)/1000.0 AS [Total Worker Time (ms)],
(qs.total_worker_time/qs.execution_count)/1000.0 AS [Avg Worker Time (ms)],
(qs.total_elapsed_time)/1000.0 AS [Total Elapsed Time (ms)],
(qs.total_elapsed_time/qs.execution_count)/1000.0 AS [Avg Elapsed Time (ms)],
qs.creation_time AS [Creation Time]
,t.text AS [Complete Query Text], qp.query_plan AS [Query Plan]
FROM sys.dm_exec_query_stats AS qs WITH (NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS t
CROSS APPLY sys.dm_exec_query_plan(plan_handle) AS qp
WHERE t.dbid = DB_ID()
ORDER BY qs.execution_count DESC OPTION (RECOMPILE);-- frequently ran query
-- ORDER BY [Total Logical Reads (MB)] DESC OPTION (RECOMPILE);-- High Disk Reading query
-- ORDER BY [Avg Worker Time (ms)] DESC OPTION (RECOMPILE);-- High CPU query
-- ORDER BY [Avg Elapsed Time (ms)] DESC OPTION (RECOMPILE);-- Long Running query

linkedin.com/in/teddy-t-28a76b200

2. Script: Captures System Memory Usage

--Works On: 2008, 2008 R2, 2012, 2014, 2016
/*********************************************************************/
select
      total_physical_memory_kb/1024 AS total_physical_memory_mb,
      available_physical_memory_kb/1024 AS available_physical_memory_mb,
      total_page_file_kb/1024 AS total_page_file_mb,
      available_page_file_kb/1024 AS available_page_file_mb,
      100 - (100 * CAST(available_physical_memory_kb AS DECIMAL(18,3))/CAST(total_physical_memory_kb AS DECIMAL(18,3))) 
      AS 'Percentage_Used',
      system_memory_state_desc
from  sys.dm_os_sys_memory;

linkedin.com/in/teddy-t-28a76b200

3.  Script: SQL Server Process Memory Usage
-- Works On: 2008, 2008 R2, 2012, 2014, 2016
/**************************************************************/
select
      physical_memory_in_use_kb/1048576.0 AS 'physical_memory_in_use (GB)',
      locked_page_allocations_kb/1048576.0 AS 'locked_page_allocations (GB)',
      virtual_address_space_committed_kb/1048576.0 AS 'virtual_address_space_committed (GB)',
      available_commit_limit_kb/1048576.0 AS 'available_commit_limit (GB)',
      page_fault_count as 'page_fault_count'
from  sys.dm_os_process_memory;

linkedin.com/in/teddy-t-28a76b200


4. Script: Database Wise Buffer Usage

--Works On: 2008, 2008 R2, 2012, 2014, 2016
/**************************************************************/
DECLARE @total_buffer INT;
SELECT  @total_buffer = cntr_value 
FROM   sys.dm_os_performance_counters
WHERE  RTRIM([object_name]) LIKE '%Buffer Manager' 
       AND counter_name = 'Database Pages';
;WITH DBBuffer AS
(
SELECT  database_id,
        COUNT_BIG(*) AS db_buffer_pages,
        SUM (CAST ([free_space_in_bytes] AS BIGINT)) / (1024 * 1024) AS [MBEmpty]
FROM    sys.dm_os_buffer_descriptors
GROUP BY database_id
)
SELECT
       CASE [database_id] WHEN 32767 THEN 'Resource DB' ELSE DB_NAME([database_id]) END AS 'db_name',
       db_buffer_pages AS 'db_buffer_pages',
       db_buffer_pages / 128 AS 'db_buffer_Used_MB',
       [mbempty] AS 'db_buffer_Free_MB',
       CONVERT(DECIMAL(6,3), db_buffer_pages * 100.0 / @total_buffer) AS 'db_buffer_percent'
FROM   DBBuffer
ORDER BY db_buffer_Used_MB DESC;

linkedin.com/in/teddy-t-28a76b200

5. Memory or RAM usage check script
--
select
(physical_memory_in_use_kb/1024)Phy_Memory_usedby_Sqlserver_MB,
(locked_page_allocations_kb/1024 )Locked_pages_used_Sqlserver_MB,
(virtual_address_space_committed_kb/1024 )Total_Memory_UsedBySQLServer_MB,
process_physical_memory_low,
process_virtual_memory_low
from sys. dm_os_process_memory

linkedin.com/in/teddy-t-28a76b200

6. SQL Query to check buffer catch hit ratio
--
SELECT counter_name as CounterName, (a.cntr_value * 1.0 / b.cntr_value) * 100.0 as BufferCacheHitRatio
FROM sys.dm_os_performance_counters  a JOIN  (SELECT cntr_value,OBJECT_NAME FROM sys.dm_os_performance_counters
WHERE counter_name = 'Buffer cache hit ratio base' AND OBJECT_NAME LIKE '%Buffer Manager%') b ON  a.OBJECT_NAME = b.OBJECT_NAME 
WHERE a.counter_name =
'Buffer cache hit ratio' AND a.OBJECT_NAME LIKE '%Buffer Manager%

linkedin.com/in/teddy-t-28a76b200


7. Extract to perform counters to a temporary table  

--USE THIS SCRIPT TO MONITOR SQL SERVER MEMEORY USAGE :TOP PERFORMANCE COUNTERS-MEMORY: Jan 03/2021, David Dere
-- Get size of SQL Server Page in bytes
DECLARE @pg_size INT, @Instancename varchar(50)
SELECT @pg_size = low from master..spt_values where number = 1 and type = 'E'
-- Extract perfmon counters to a temporary table
IF OBJECT_ID('tempdb..#perfmon_counters') is not null DROP TABLE #perfmon_counters
SELECT * INTO #perfmon_counters FROM sys.dm_os_performance_counters;
-- Get SQL Server instance name as it require for capturing Buffer Cache hit Ratio
SELECT  @Instancename = LEFT([object_name], (CHARINDEX(':',[object_name]))) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Buffer cache hit ratio';
SELECT * FROM (
SELECT  'Total Server Memory (GB)' as Cntr,
        (cntr_value/1048576.0) AS Value 
FROM    #perfmon_counters 
WHERE   counter_name = 'Total Server Memory (KB)'
UNION ALL
SELECT  'Target Server Memory (GB)', 
        (cntr_value/1048576.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Target Server Memory (KB)'
UNION ALL
SELECT  'Connection Memory (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Connection Memory (KB)'
UNION ALL
SELECT  'Lock Memory (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Lock Memory (KB)'
UNION ALL
SELECT  'SQL Cache Memory (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'SQL Cache Memory (KB)'
UNION ALL
SELECT  'Optimizer Memory (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Optimizer Memory (KB) '
UNION ALL
SELECT  'Granted Workspace Memory (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Granted Workspace Memory (KB) '
UNION ALL
SELECT  'Cursor memory usage (MB)', 
        (cntr_value/1024.0) 
FROM    #perfmon_counters 
WHERE   counter_name = 'Cursor memory usage' and instance_name = '_Total'
UNION ALL
SELECT  'Total pages Size (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name= @Instancename+'Buffer Manager' 
        and counter_name = 'Total pages'
UNION ALL
SELECT  'Database pages (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name = @Instancename+'Buffer Manager' and counter_name = 'Database pages'
UNION ALL
SELECT  'Free pages (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name = @Instancename+'Buffer Manager' 
        and counter_name = 'Free pages'
UNION ALL
SELECT  'Reserved pages (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Reserved pages'
UNION ALL
SELECT  'Stolen pages (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Stolen pages'
UNION ALL
SELECT  'Cache Pages (MB)', 
        (cntr_value*@pg_size)/1048576.0 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Plan Cache' 
        and counter_name = 'Cache Pages' and instance_name = '_Total'
UNION ALL
SELECT  'Page Life Expectency in seconds',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Page life expectancy'
UNION ALL
SELECT  'Free list stalls/sec',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Free list stalls/sec'
UNION ALL
SELECT  'Checkpoint pages/sec',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Checkpoint pages/sec'
UNION ALL
SELECT  'Lazy writes/sec',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Buffer Manager' 
        and counter_name = 'Lazy writes/sec'
UNION ALL
SELECT  'Memory Grants Pending',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Memory Manager' 
        and counter_name = 'Memory Grants Pending'
UNION ALL
SELECT  'Memory Grants Outstanding',
        cntr_value 
FROM    #perfmon_counters 
WHERE   object_name=@Instancename+'Memory Manager' 
        and counter_name = 'Memory Grants Outstanding'
UNION ALL
SELECT  'process_physical_memory_low',
        process_physical_memory_low 
FROM    sys.dm_os_process_memory WITH (NOLOCK)
UNION ALL
SELECT  'process_virtual_memory_low',
        process_virtual_memory_low 
FROM    sys.dm_os_process_memory WITH (NOLOCK)
UNION ALL
SELECT  'Max_Server_Memory (MB)' ,
        [value_in_use] 
FROM    sys.configurations 
WHERE   [name] = 'max server memory (MB)'
UNION ALL
SELECT  'Min_Server_Memory (MB)' ,
        [value_in_use] 
FROM    sys.configurations 
WHERE   [name] = 'min server memory (MB)'
UNION ALL
SELECT  'BufferCacheHitRatio',
        (a.cntr_value * 1.0 / b.cntr_value) * 100.0 
FROM    sys.dm_os_performance_counters a
        JOIN (SELECT cntr_value,OBJECT_NAME FROM sys.dm_os_performance_counters
              WHERE counter_name = 'Buffer cache hit ratio base' AND 
                    OBJECT_NAME = @Instancename+'Buffer Manager') b ON 
                    a.OBJECT_NAME = b.OBJECT_NAME WHERE a.counter_name = 'Buffer cache hit ratio' 
                    AND a.OBJECT_NAME = @Instancename+'Buffer Manager'
) AS D;

linkedin.com/in/teddy-t-28a76b200



8. To check the total server memory
--- 
dbcc memorystatus




9. CPU Utilization History 

--

/* CPU Utilization History (last 144 minutes in one minute intervals) */
DECLARE @ts_now BIGINT = ( SELECT
							cpu_ticks / ( cpu_ticks / ms_ticks )
							FROM
							sys.dm_os_sys_info
							) ; 
SELECT TOP ( 144 )
	@@SERVERNAME AS [Server Name],
	SQLProcessUtilization AS [SQL Server Process CPU Utilization] ,
	SystemIdle AS [System Idle Process] ,
	100 - SystemIdle - SQLProcessUtilization AS [Other Process CPU Utilization] ,
	DATEADD(ms, -1 * ( @ts_now - [timestamp] ), CURRENT_TIMESTAMP) AS [Event Time],
	CURRENT_TIMESTAMP AS [Collection Time]
FROM
	( SELECT
		record.value('(./Record/@id)[1]', 'INT') AS record_id ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]',
						'INT') AS [SystemIdle] ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
						'INT') AS [SQLProcessUtilization] ,
		[timestamp]
		FROM
		( SELECT
			[timestamp] ,
			CONVERT(XML, record) AS [record]
			FROM
			sys.dm_os_ring_buffers
			WHERE
			ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
			AND record LIKE N'%<SystemHealth>%'
		) AS x
	) AS y
ORDER BY
	record_id DESC
OPTION
	(RECOMPILE);

10. CPU Utilization History  (last 144 minutes in one minute intervals)

SET ANSI_NULLS, QUOTED_IDENTIFIER ON;
GO
DECLARE @Duration INT=240-------------You can put here Duration like 60/120
DECLARE @ts_now BIGINT = ( SELECT
cpu_ticks / ( cpu_ticks / ms_ticks )
FROM
sys.dm_os_sys_info
) ;
SELECT TOP ( @Duration )
@@SERVERNAME AS [Server Name],
SQLProcessUtilization AS [SQL Server Process CPU Utilization] ,
SystemIdle AS [System Idle Process] ,
100 - SystemIdle - SQLProcessUtilization AS [Other Process CPU Utilization] ,
DATEADD(ms, -1 * ( @ts_now - [timestamp] ), CURRENT_TIMESTAMP) AS [Event Time],
CURRENT_TIMESTAMP AS [Collection Time]
FROM
( SELECT
record.value('(./Record/@id)[1]', 'INT') AS record_id ,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]',
'INT') AS [SystemIdle] ,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
'INT') AS [SQLProcessUtilization] ,
[timestamp]
FROM
( SELECT
[timestamp] ,
CONVERT(XML, record) AS [record]
FROM
sys.dm_os_ring_buffers
WHERE
ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
AND record LIKE N'%<SystemHealth>%'
) AS x
) AS y
ORDER BY
record_id DESC
OPTION
(RECOMPILE);

11. Check which is using more memory

Select * from sys.sysprocesses where cpu > 100000
select * from sys.sysprocesses where memusage > xxxxx value
---Check which is using more memory and cpu
---Take that session id and put it in DBCC INPUTBUFFER  (SESSION ID)


12. Check memory usage 

select
(physical_memory_in_use_kb/1024)Phy_Memory_usedby_Sqlserver_MB,
(locked_page_allocations_kb/1024 )Locked_pages_used_Sqlserver_MB,
(virtual_address_space_committed_kb/1024 )Total_Memory_UsedBySQLServer_MB,
process_physical_memory_low,
process_virtual_memory_low
from sys. dm_os_process_memory

13. Check CPU by Database

/* CPU by Database */
WITH DB_CPU_Stats
AS
(SELECT DatabaseID, DB_Name(DatabaseID) AS [DatabaseName], SUM(total_worker_time) AS [CPU_Time_Ms]
 FROM sys.dm_exec_query_stats AS qs
 CROSS APPLY (SELECT CONVERT(INT, value) AS [DatabaseID] 
              FROM sys.dm_exec_plan_attributes(qs.plan_handle)
              WHERE attribute = N'dbid') AS F_DB
 GROUP BY DatabaseID)
SELECT @@SERVERNAME AS [Server Name], ROW_NUMBER() OVER(ORDER BY [CPU_Time_Ms] DESC) AS [Row Number],
       DatabaseName AS [Database Name], [CPU_Time_Ms] AS [CPU Time MS], 
       CAST([CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPU Percent], 
	  CURRENT_TIMESTAMP AS [Collection Time] 
FROM DB_CPU_Stats
WHERE DatabaseID <> 32767 -- ResourceDB
ORDER BY [Row Number] OPTION (RECOMPILE);




14. Captures System Memory Usage


15. /*********************************************************************/
--Script: Captures System Memory Usage
--Works On: 2008, 2008 R2, 2012, 2014, 2016
/*********************************************************************/
select
      total_physical_memory_kb/1024 AS total_physical_memory_mb,
      available_physical_memory_kb/1024 AS available_physical_memory_mb,
      total_page_file_kb/1024 AS total_page_file_mb,
      available_page_file_kb/1024 AS available_page_file_mb,
      100 - (100 * CAST(available_physical_memory_kb AS DECIMAL(18,3))/CAST(total_physical_memory_kb AS DECIMAL(18,3))) 
      AS 'Percentage_Used',
      system_memory_state_desc
from  sys.dm_os_sys_memory;


16. Check buffer catch hit Ratio

SELECT counter_name as CounterName, (a.cntr_value * 1.0 / b.cntr_value) * 100.0 as BufferCacheHitRatio FROM sys.dm_os_performance_counters  a JOIN  (SELECT cntr_value,OBJECT_NAME FROM sys.dm_os_performance_counters WHERE counter_name = 'Buffer cache hit ratio base' AND OBJECT_NAME LIKE '%Buffer Manager%') b ON  a.OBJECT_NAME = b.OBJECT_NAME WHERE a.counter_name =
'Buffer cache hit ratio' AND a.OBJECT_NAME LIKE '%Buffer Manager%'


17. SQL Server Version and Operating System Version Information

/* SQL Server Version and Operating System Version Information */
SELECT 
	 @@SERVERNAME AS [Server Name]
	,REPLACE(REPLACE(REPLACE(@@VERSION, CHAR(10), CHAR(32)), CHAR(13), CHAR(32)), CHAR(9), CHAR(32)) AS [SQL Server and OS Version Info]
	,(SELECT create_date FROM sys.databases WHERE database_id = 2) AS [Last SQL Server Restart]
	,CURRENT_TIMESTAMP AS [Collection Time];

18. SQL Server Property
SELECT 
	SERVERPROPERTY('MachineName') AS [Machine Name], 
	SERVERPROPERTY('ServerName') AS [Server Name],  
	SERVERPROPERTY('InstanceName') AS [Instance], 
	SERVERPROPERTY('IsClustered') AS [Is Clustered], 
	SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS [Computer Name Physical NetBIOS], 
	SERVERPROPERTY('Edition') AS [Edition], 
	SERVERPROPERTY('ProductLevel') AS [Product Level],		
	SERVERPROPERTY('ProductUpdateLevel') AS [Product Update Level],	
	SERVERPROPERTY('ProductVersion') AS [Product Version],
	SERVERPROPERTY('ProductMajorVersion') AS [Product Major Version], 
	SERVERPROPERTY('ProductMinorVersion') AS [Product Minor Version], 
	SERVERPROPERTY('ProductBuild') AS [Product Build], 
	SERVERPROPERTY('ProductBuildType') AS [Product Build Type],		    
	SERVERPROPERTY('ProductUpdateReference') AS [Product Update Reference],
	SERVERPROPERTY('ProcessID') AS [ProcessID],
	SERVERPROPERTY('Collation') AS [Collation], 
	SERVERPROPERTY('IsFullTextInstalled') AS [Is Full Text Installed], 
	SERVERPROPERTY('IsIntegratedSecurityOnly') AS [Is Integrated Security Only],
	SERVERPROPERTY('FilestreamConfiguredLevel') AS [Filestream Configured Level],
	SERVERPROPERTY('IsHadrEnabled') AS [Is Hadr Enabled], 
	SERVERPROPERTY('HadrManagerStatus') AS [Hadr Manager Status],
	SERVERPROPERTY('IsXTPSupported') AS [Is XTP Supported],
	SERVERPROPERTY('InstanceDefaultDataPath') AS [Instance Default Data Path],
	SERVERPROPERTY('InstanceDefaultLogPath') AS [Instance Default Log Path],
	SERVERPROPERTY('BuildClrVersion') AS [Build CLR Version],
	CURRENT_TIMESTAMP AS [Collection Time];





19. SQL Server Installation Data

SELECT 
	SERVERPROPERTY('MachineName') AS [Machine Name], 
	SERVERPROPERTY('ServerName') AS [Server Name],  
	SERVERPROPERTY('InstanceName') AS [Instance], 
	SERVERPROPERTY('IsClustered') AS [Is Clustered], 
	SERVERPROPERTY('ComputerNamePhysicalNetBIOS') AS [Computer Name Physical NetBIOS], 
	SERVERPROPERTY('Edition') AS [Edition], 
	SERVERPROPERTY('ProductLevel') AS [Product Level],		
	SERVERPROPERTY('ProductUpdateLevel') AS [Product Update Level],	
	SERVERPROPERTY('ProductVersion') AS [Product Version],
	SERVERPROPERTY('ProductMajorVersion') AS [Product Major Version], 
	SERVERPROPERTY('ProductMinorVersion') AS [Product Minor Version], 
	SERVERPROPERTY('ProductBuild') AS [Product Build], 
	SERVERPROPERTY('ProductBuildType') AS [Product Build Type],		    
	SERVERPROPERTY('ProductUpdateReference') AS [Product Update Reference],
	SERVERPROPERTY('ProcessID') AS [ProcessID],
	SERVERPROPERTY('Collation') AS [Collation], 
	SERVERPROPERTY('IsFullTextInstalled') AS [Is Full Text Installed], 
	SERVERPROPERTY('IsIntegratedSecurityOnly') AS [Is Integrated Security Only],
	SERVERPROPERTY('FilestreamConfiguredLevel') AS [Filestream Configured Level],
	SERVERPROPERTY('IsHadrEnabled') AS [Is Hadr Enabled], 
	SERVERPROPERTY('HadrManagerStatus') AS [Hadr Manager Status],
	SERVERPROPERTY('IsXTPSupported') AS [Is XTP Supported],
	SERVERPROPERTY('InstanceDefaultDataPath') AS [Instance Default Data Path],
	SERVERPROPERTY('InstanceDefaultLogPath') AS [Instance Default Log Path],
	SERVERPROPERTY('BuildClrVersion') AS [Build CLR Version],
	CURRENT_TIMESTAMP AS [Collection Time];

20. SQL Server Registry Information 

/* SQL Server Registry Information */
SELECT @@SERVERNAME AS [Server Name], registry_key AS [Registry Key], value_name AS [Value Name], value_data AS [Value Data], CURRENT_TIMESTAMP AS [Collection Time] 
FROM sys.dm_server_registry WITH (NOLOCK) OPTION (RECOMPILE);

21. SQL server Memory Dumps


/* SQL Server Memory Dumps */
SELECT @@SERVERNAME AS [Server Name], [filename] AS [File Name], creation_time AS [Creation Time], 
size_in_bytes AS [Size in Bytes], CURRENT_TIMESTAMP AS [Collection Time] 
FROM sys.dm_server_memory_dumps WITH (NOLOCK) OPTION (RECOMPILE);



22. Windows Information

/* Windows Information */
SELECT @@SERVERNAME AS [Server Name], windows_release AS [Windows Release], windows_service_pack_level AS [Windows Service Pack Level], 
       windows_sku AS [Windows SKU], os_language_version AS [OS Language Version], CURRENT_TIMESTAMP AS [Collection Time] 
FROM sys.dm_os_windows_info OPTION (RECOMPILE);


23. Processor name and Description 

/* Processor Description */
CREATE TABLE #RDXResults
(value VARCHAR(255), data VARCHAR(255));
INSERT INTO #RDXResults
EXEC xp_instance_regread 'HKEY_LOCAL_MACHINE','HARDWARE\DESCRIPTION\System\CentralProcessor\0','ProcessorNameString';
SELECT @@SERVERNAME AS [Server Name], value AS [Value Name], data AS [Processor Information], CURRENT_TIMESTAMP AS [Collection Time] FROM #RDXResults;
DROP TABLE #RDXResults;


24. Hardware Information

SELECT @@SERVERNAME AS [Server Name], cpu_count AS [Logical CPU Count], hyperthread_ratio AS 
[Hyperthread Ratio],
cpu_count/hyperthread_ratio AS [Physical CPU Count], 
physical_memory_kb/1024 AS [Physical Memory (MB)], 
sqlserver_start_time AS [SQL Server Start Time], affinity_type_desc AS [Affinity Type Description], 
virtual_machine_type_desc AS [Virtual Machine Type], CURRENT_TIMESTAMP AS [Collection Time]
FROM sys.dm_os_sys_info OPTION (RECOMPILE);

25. System Manufacture

 System Manufacturer and Model Number */
CREATE TABLE #RDXResults
	(logdate DATETIME, processinfo VARCHAR(255), [Text] VARCHAR(255));
INSERT INTO #RDXResults
EXEC xp_readerrorlog 0,1,"Manufacturer"; 
SELECT @@SERVERNAME AS [Server Name], logdate AS [Log Date], processinfo AS [Process Info], [Text] AS [System Manufacturer], CURRENT_TIMESTAMP AS [Collection Time] FROM #RDXResults;
DROP TABLE #RDXResults;


26. Instant File Initialization is Enabled


/* Instant File Initialization is Enabled */
CREATE TABLE #RDXResults
	([Output] VARCHAR(255));
begin try
INSERT INTO #RDXResults
EXEC xp_cmdshell 'whoami /priv';
end try
begin catch
INSERT INTO #RDXResults
SELECT 'Error occured. Please manually check via local security policy. '  as [Output]
end catch
SELECT @@SERVERNAME AS [Server Name], [Output] AS [Privileges Information], CURRENT_TIMESTAMP AS [Collection Time]
FROM #RDXResults WHERE [output] IS NOT NULL;
DROP TABLE #RDXResults;

linkedin.com/in/teddy-t-28a76b200




27. How to Enable and Disable Login in SQL Server - SQL Server DBA Tutorial?

--How to enable SQL Server Login

 USE [master]
GO 
GRANT CONNECT SQL TO [Techbrothers]
GO 
ALTER LOGIN [Techbrothers] ENABLE
GO

 -- How to disable SQL Server Login 

USE [master]
GO 
DENY CONNECT SQL TO [Techbrothers]
GO
 ALTER LOGIN [Techbrothers] DISABLE
GO


28. System Memory Information

/* System Memory Information */
/* Checks for external memory pressure */
SELECT @@SERVERNAME AS [Server Name], total_physical_memory_kb, available_physical_memory_kb, 
       total_page_file_kb, available_page_file_kb, system_memory_state_desc, CURRENT_TIMESTAMP AS [Collection Time]
FROM sys.dm_os_sys_memory OPTION (RECOMPILE);


29. Process Memory

/* SQL Server Process Address Space Information  */ 
/* Checks for internal memory pressure */
SELECT @@SERVERNAME AS [Server Name], physical_memory_in_use_kb,locked_page_allocations_kb, 
       page_fault_count, memory_utilization_percentage, 
       available_commit_limit_kb, process_physical_memory_low, 
       process_virtual_memory_low, CURRENT_TIMESTAMP AS [Collection Time]
FROM sys.dm_os_process_memory OPTION (RECOMPILE);

27.1   Max vs Min Memory Configuration 

sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO

sp_configure 'min server memory', 2048;
GO
RECONFIGURE;
GO
 

sp_configure 'max server memory', 4096;
GO
RECONFIGURE;
GO


30. SQL Server Ring Buffer Memory Warnings 


/* SQL Server Ring Buffer Memory Warnings */
WITH     RingBuffer
AS       (SELECT CAST (dorb.record AS XML) AS xRecord,
                 dorb.timestamp
          FROM   sys.dm_os_ring_buffers AS dorb
          WHERE  dorb.ring_buffer_type = 'RING_BUFFER_RESOURCE_MONITOR')
SELECT   @@SERVERNAME AS [Server Name],
	 xr.value('(ResourceMonitor/Notification)[1]', 'VARCHAR(75)') AS RmNotification,
         xr.value('(ResourceMonitor/IndicatorsProcess)[1]', 'tinyint') AS IndicatorsProcess,
         xr.value('(ResourceMonitor/IndicatorsSystem)[1]', 'tinyint') AS IndicatorsSystem,
         DATEADD (ss, (-1 * ((dosi.cpu_ticks / CONVERT (float, ( dosi.cpu_ticks / dosi.ms_ticks ))) - [timestamp])/1000), CURRENT_TIMESTAMP) AS RmDateTime,
         xr.value('(MemoryNode/TargetMemory)[1]', 'BIGINT') AS TargetMemory,
         xr.value('(MemoryNode/ReserveMemory)[1]', 'BIGINT') AS ReserveMemory,
         xr.value('(MemoryNode/CommittedMemory)[1]', 'BIGINT') AS CommitedMemory,
         xr.value('(MemoryNode/SharedMemory)[1]', 'BIGINT') AS SharedMemory,
         xr.value('(MemoryNode/PagesMemory)[1]', 'BIGINT') AS PagesMemory,
         xr.value('(MemoryRecord/MemoryUtilization)[1]', 'BIGINT') AS MemoryUtilization,
         xr.value('(MemoryRecord/TotalPhysicalMemory)[1]', 'BIGINT') AS TotalPhysicalMemory,
         xr.value('(MemoryRecord/AvailablePhysicalMemory)[1]', 'BIGINT') AS AvailablePhysicalMemory,
         xr.value('(MemoryRecord/TotalPageFile)[1]', 'BIGINT') AS TotalPageFile,
         xr.value('(MemoryRecord/AvailablePageFile)[1]', 'BIGINT') AS AvailablePageFile,
         xr.value('(MemoryRecord/TotalVirtualAddressSpace)[1]', 'BIGINT') AS TotalVirtualAddressSpace,
         xr.value('(MemoryRecord/AvailableVirtualAddressSpace)[1]', 'BIGINT') AS AvailableVirtualAddressSpace,
         xr.value('(MemoryRecord/AvailableExtendedVirtualAddressSpace)[1]', 'BIGINT') AS AvailableExtendedVirtualAddressSpace,
    CURRENT_TIMESTAMP AS [Collection Time]
FROM     RingBuffer AS rb CROSS APPLY rb.xRecord.nodes ('Record') AS record(xr) CROSS JOIN sys.dm_os_sys_info AS dosi
ORDER BY RmDateTime DESC;




31. How to Enable Query Store option of a databases in SQL Server 2016? 

USE [master]
GO
ALTER DATABASE [TSQL2012] SET QUERY_STORE = ON
GO
ALTER DATABASE [TSQL2012] SET QUERY_STORE (OPERATION_MODE = READ_WRITE)
GO

32. Best script to What is going on my Server I.e. CMD, BLOCK, MEMORY, CPU.


SELECT r.session_id, 
s.program_name, 
s.login_name, 
r.start_time, 
r.status, 
r.command, 
Object_name(sqltxt.objectid, sqltxt.dbid) AS ObjectName, 
Substring(sqltxt.text, ( r.statement_start_offset / 2 ) + 1, ( ( 
CASE r.statement_end_offset 
WHEN -1 THEN 
datalength(sqltxt.text) 
ELSE r.statement_end_offset 
END 
- r.statement_start_offset ) / 2 ) + 1) AS active_statement,
r.percent_complete, 
Db_name(r.database_id) AS DatabaseName, 
r.blocking_session_id, 
r.wait_time, 
r.wait_type, 
r.wait_resource, 
r.open_transaction_count, 
r.cpu_time,-- in milli sec 
r.reads, 
r.writes, 
r.logical_reads, 
r.row_count, 
r.prev_error, 
r.granted_query_memory, 
Cast(sqlplan.query_plan AS XML) AS QueryPlan, 
CASE r.transaction_isolation_level 
WHEN 0 THEN 'Unspecified' 
WHEN 1 THEN 'ReadUncomitted' 
WHEN 2 THEN 'ReadCommitted' 
WHEN 3 THEN 'Repeatable' 
WHEN 4 THEN 'Serializable' 
WHEN 5 THEN 'Snapshot' 
END AS Issolation_Level, 
r.sql_handle, 
r.plan_handle 
FROM sys.dm_exec_requests r WITH (nolock) 
INNER JOIN sys.dm_exec_sessions s WITH (nolock) 
ON r.session_id = s.session_id 
CROSS apply sys.Dm_exec_sql_text(r.sql_handle) sqltxt 
CROSS apply 
sys.Dm_exec_text_query_plan(r.plan_handle, r.statement_start_offset, r.statement_end_offset) sqlplan
WHERE r.status <> 'background' 
ORDER BY r.session_id 
go 
Select * from sys.sysprocesses where cpu > 100000
select * from sys.sysprocesses where memusage > xxxxx value
---Check which is using more memory and cpu
---Take that session id and put it in DBCC INPUTBUFFER  (SESSION ID


USE [master];
GO
SELECT r.session_id, r.blocking_session_id,
 r.wait_type, r.wait_time,
 r.wait_resource, r.transaction_isolation_level,
 r.[lock_timeout], st.[text] AS BlockedSQLStatement
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st
WHERE blocking_session_id <> 0;
GO


33. Check Database Size

SELECT name AS [File Name], 
        physical_name AS [Physical Name], 
        size/128.0 AS [Total Size in MB], 
        size/128.0 - CAST(FILEPROPERTY(name, 'SpaceUsed') AS int)/128.0 AS [Available Space In MB], 
        [growth], [file_id]
        FROM sys.database_files
        WHERE type_desc = 'LOG' 
DBCC SQLPERF (logspace)


34. Check Database status Online and Offline 

SELECT NAME, STATE_DESC
FROM SYS.DATABASES



35. Database Fille Location Database state that is Online or Offline and Fille state that is Offline and online 

/* Database File Location */
SELECT
	@@SERVERNAME AS [Server Name],
    db.name AS [Database Name],
	db.state_desc AS [Database State],
    mf.[file_id] AS [File ID],
    mf.name AS [Name],
    mf.physical_name AS [Physical Name],
    mf.type_desc AS [Type Description],
    mf.state_desc AS [File State],
    mf.SIZE AS [Number of 8K Pages],
    CONVERT(BIGINT, mf.size / 128.0) AS [Total Size in MB],
	CURRENT_TIMESTAMP AS [Collection Time] 
FROM sys.master_files AS mf
	LEFT JOIN sys.databases AS db WITH (NOLOCK) ON mf.database_id = db.database_id
ORDER BY db.name
OPTION (RECOMPILE);

36. Database Backup Check by Hour
 
use msdb;
select @@SERVERNAME AS server_name, S.backup_set_id,s.user_name,s.first_lsn,s.last_lsn,s.backup_start_date,s.backup_finish_date,s.type,s.database_name,F.physical_device_name from backupset S
inner join backupmediafamily F on F.Media_set_id = S.backup_set_id 
where s.backup_start_date > DATEADD(HH,-12,GETDATE())
and s.type='d'


37. Fetch job failed from SQL server 


SELECT sj.[name] AS "Failed Job Name", sh.run_date,sh.[message]
FROM msdb.dbo.sysjobhistory sh INNER JOIN msdb.dbo.sysjobs sj ON sj.job_id =
sh.job_id
WHERE sh.run_status=0 AND DATEDIFF(dd,cast (cast (sh.run_date AS VARCHAR(20)) AS
DATE),GETDATE())=0
GROUP BY sj.[name],sh.run_date,sh.message
Go


38. Fetch SQL server Agent Job Failure / XXX 

USE DBA
GO
SELECT * FROM [dbo].[sql_server_agent_job_failure]


39. Check Daily Backup
/* Full DB Backup*/
use msdb;
select @@SERVERNAME AS server_name,s.database_name, s.backup_start_date,s.backup_finish_date
FROM backupset S
inner join backupmediafamily F on F.Media_set_id = S.backup_set_id 
where s.backup_start_date > DATEADD(HH,-10,GETDATE())
and s.type='D'
ORDER BY 2
--------------------------------------------------------------------------------
/* Logshipping and full DB backup*/
SELECT d.[Name] AS DatabaseName,
LastFullBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
bs.[database_name]=d.[name] AND bs.[type]='D')
--LastDiffBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
--bs.[database_name]=d.[name] AND bs.[type]='I'),
--LastLogBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
--bs.[database_name]=d.[name] AND bs.[type]='L')
FROM sys.databases d
ORDER BY 1;
GO
-------------------------------------------------------------------------------------------------------------
/*Logshipping copy restore, run on secondary server*/
use msdb
select primary_server,primary_database,secondary_server,
secondary_database,last_copied_date,last_copied_file,last_restored_date,
last_restored_file
FROM dbo.log_shipping_monitor_secondary

40.  List of Job Failures

use msdb
go
select h.server as [Server],
j.[name] as [Name],
h.message as [Message],
h.run_date as LastRunDate, 
h.run_time as LastRunTime
from sysjobhistory h
inner join sysjobs j on h.job_id = j.job_id
where j.enabled = 1 
and h.instance_id in
(select max(h.instance_id)
from sysjobhistory h group by (h.job_id))
and h.run_status = 0


41. Recent Full backup Detail
/* Recent full backup details */
CREATE TABLE #rdxresults
	(
		[Server Name] VARCHAR(255),
		[Database Name] VARCHAR(255),
		[Physical Name] VARCHAR (2500),
		[Uncompressed Backup Size (MB)] BIGINT,
		[Compressed Backup Size (MB)] BIGINT,
		[Compression Ratio] NUMERIC(20,2),
		[Backup Elapsed Time (sec)] INT,
		[Backup Finish Date] DATETIME,
		[Collection Time] DATETIME
	);
INSERT INTO #rdxresults
EXEC sp_MSforeachdb @command1 = 'USE [?];
SELECT bs.server_name AS [Server Name], bs.database_name AS [Database Name], bf.physical_device_name as [Physical Name],
CONVERT (BIGINT, bs.backup_size / 1048576 ) AS [Uncompressed Backup Size (MB)],
CONVERT (BIGINT, bs.compressed_backup_size / 1048576 ) AS [Compressed Backup Size (MB)],
CONVERT (NUMERIC (20,2), (CONVERT (FLOAT, bs.backup_size) /
CONVERT (FLOAT, bs.compressed_backup_size))) AS [Compression Ratio], 
DATEDIFF (SECOND, bs.backup_start_date, bs.backup_finish_date) AS [Backup Elapsed Time (sec)],
bs.backup_finish_date AS [Backup Finish Date],
CURRENT_TIMESTAMP AS [Collection Time]
FROM msdb.dbo.backupset AS bs WITH (NOLOCK)
inner join msdb.dbo.backupmediafamily bf on bf.media_set_id = bs.media_set_id
WHERE DATEDIFF (SECOND, bs.backup_start_date, bs.backup_finish_date) > 0 
AND bs.backup_size > 0
AND bs.type = ''D'' -- Change to L if you want Log backups
AND database_name = DB_NAME(DB_ID())
AND bs.backup_finish_date >= CONVERT(CHAR(8), (SELECT DATEADD (DAY,(-90), GETDATE())), 112)
ORDER BY bs.database_name, bs.backup_finish_date DESC OPTION (RECOMPILE);';
SELECT * FROM #rdxresults ORDER BY [Database Name] ASC, [Backup Finish Date] ASC;
DROP TABLE #rdxresults;

42. SCRIPT TO CHECK AND SEE THE N# OF CORES AND CPU OF SQL SERVER

DECLARE @xp_msver TABLE (
    [idx] [int] NULL
    ,[c_name] [varchar](100) NULL
    ,[int_val] [float] NULL
    ,[c_val] [varchar](128) NULL
    )
 
INSERT INTO @xp_msver
EXEC ('[master]..[xp_msver]');;
 
WITH [ProcessorInfo]
AS (
    SELECT ([cpu_count] / [hyperthread_ratio]) AS [number_of_physical_cpus]
        ,CASE
            WHEN hyperthread_ratio = cpu_count
                THEN cpu_count
            ELSE (([cpu_count] - [hyperthread_ratio]) / ([cpu_count] / [hyperthread_ratio]))
            END AS [number_of_cores_per_cpu]
        ,CASE
            WHEN hyperthread_ratio = cpu_count
                THEN cpu_count
            ELSE ([cpu_count] / [hyperthread_ratio]) * (([cpu_count] - [hyperthread_ratio]) / ([cpu_count] / [hyperthread_ratio]))
            END AS [total_number_of_cores]
        ,[cpu_count] AS [number_of_virtual_cpus]
        ,(
            SELECT [c_val]
            FROM @xp_msver
            WHERE [c_name] = 'Platform'
            ) AS [cpu_category]
    FROM [sys].[dm_os_sys_info]
    )
SELECT [number_of_physical_cpus]
    ,[number_of_cores_per_cpu]
    ,[total_number_of_cores]
    ,[number_of_virtual_cpus]
    ,LTRIM(RIGHT([cpu_category], CHARINDEX('x', [cpu_category]) - 1)) AS [cpu_category]
FROM [ProcessorInfo]


43. SCRIPT TO CHECK DB GROWTH BT MONTH LAST MONTH AND CURRENT MONTH DATABASE GROWTH

/*
SCRIPT PURPOSE TO FETCH DIFFERENCE LAST MONTH AND CURRENT MONTH DATABASE GROWTH IN PERCENTAGE.
*/
--------------------------------------------------------------------------------
declare @startDate datetime; set @startDate = getdate(); 
----------------------------------------------------------------------------------------
DECLARE @SQL VARCHAR(4000), @TABLE_SQL VARCHAR(4000), @INDEX_SQL VARCHAR(4000)
declare @cnt bigint, @email_cnt bigint ;
declare @subject nvarchar(255) ;
declare @ip_address nvarchar(150); set @ip_address = '';
declare @tableHTML  nvarchar(max), @queryHTML  nvarchar(max);
declare @server_domain nvarchar(255); set @server_domain = '';
declare @profile_name nvarchar(128); 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
--------------------------------------------------------------------------------
if (object_id('tempdb.dbo.#db_growth') is not null) drop table #db_growth
if (object_id('tempdb.dbo.#db_backup') is not null) drop table #db_backup
----------------------------------------------------------------------------------------
CREATE TABLE #db_growth (ID BIGINT IDENTITY(1,1),
Instance_Name varchar(256), Database_Name varchar(256),Current_Month varchar(50),
Last_Month varchar(50), Growth numeric(20,3) , run_date char(10))
----------------------------------------------------------------------------------------
CREATE TABLE #db_backup 
( Database_Name varchar(256), [0] numeric(10, 1), [-1] numeric(10, 1), [-2] numeric(10, 1), [-3] numeric(10, 1))
----------------------------------------------------------------------------------------
set @SQL = ''
----------------------------------------------------------------------------------------
if (select count(1) 
                from msdb.sys.columns c inner join msdb.sys.tables t on c.object_id = t.object_id
                where t.name = 'backupfile' and c.name = 'compressed_backup_size') > 0
begin 
set @SQL = @SQL + '
SELECT bs.database_name AS database_name 
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  0 THEN CONVERT(numeric(10, 1), max(bs.compressed_backup_size/1024/1024)) ELSE 0 END AS [0] 
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -1 THEN CONVERT(numeric(10, 1), max(bs.compressed_backup_size/1024/1024)) ELSE 0 END AS [-1]
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -2 THEN CONVERT(numeric(10, 1), max(bs.compressed_backup_size/1024/1024)) ELSE 0 END AS [-2]
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -3 THEN CONVERT(numeric(10, 1), max(bs.compressed_backup_size/1024/1024)) ELSE 0 END AS [-3]
                FROM msdb.dbo.backupset as bs with(nolock)
                INNER JOIN msdb.dbo.backupfile AS bf with(nolock) ON bs.backup_set_id = bf.backup_set_id 
                WHERE bs.database_name not in (''tempdb'') 
                AND bs.type = ''D''
                AND bs.backup_start_date BETWEEN dateadd(month, -3, ''' + convert(char(26),@startDate,121) + ''') AND ''' + convert(char(26),@startDate,121) + ''' 
                group by bs.database_name ,datediff(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) 
'
end 
else 
begin
set @SQL = @SQL + '
SELECT bs.database_name AS database_name 
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  0 THEN CONVERT(numeric(10, 1), max(bs.backup_size/1024/1024)) ELSE 0 END AS [0] 
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -1 THEN CONVERT(numeric(10, 1), max(bs.backup_size/1024/1024)) ELSE 0 END AS [-1]
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -2 THEN CONVERT(numeric(10, 1), max(bs.backup_size/1024/1024)) ELSE 0 END AS [-2]
                ,CASE WHEN DATEDIFF(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) =  -3 THEN CONVERT(numeric(10, 1), max(bs.backup_size/1024/1024)) ELSE 0 END AS [-3]
                FROM msdb.dbo.backupset as bs with(nolock)
                INNER JOIN msdb.dbo.backupfile AS bf with(nolock) ON bs.backup_set_id = bf.backup_set_id 
                WHERE bs.database_name not in (''tempdb'') 
                AND bs.type = ''D''
                AND bs.backup_start_date BETWEEN dateadd(month, -3, ''' + convert(char(26),@startDate,121) + ''') AND ''' + convert(char(26),@startDate,121) + ''' 
                group by bs.database_name ,datediff(mm, ''' + convert(char(26),@startDate,121) + ''', bs.backup_start_date) 
'
end 
----------------------------------------------------------------------------------------
insert into #db_backup
exec (@SQL)
----------------------------------------------------------------------------------------
insert into #db_growth
select 
cast(isnull(serverproperty('servername'),isnull(@@servername,''))as varchar(256)) as Instance_Name,
cast(isnull(db.name,'')as varchar(256)) as Database_Name,
cast(isnull(pvt.[0],0)as varchar(50)) as Current_Month,
cast(isnull(pvt.[-1],0)as varchar(50)) as Last_Month ,
case 
                when isnull(pvt.[-1],0) = 0 then 0
                when isnull(pvt.[0],0) = 0 then 0
                when isnull(pvt.[-1],0) = isnull(pvt.[0],0) then 0
                else round((isnull(pvt.[0],0) - isnull(pvt.[-1],0)) / isnull(pvt.[-1],0) * 100,1)
end  as Growth ,
convert(char(10),getdate(),121) as run_date
from master.dbo.sysdatabases db
left outer join (
SELECT database_name, max([0])as [0], max([-1])as [-1], max([-2])as [-2], max([-3])as [-3]
FROM 
(  select * from #db_backup
    ) AS bckstat 
group by database_name    
) as pvt on db.name = pvt.database_name 
where db.name not in ('tempdb') 
order by 5 desc;
SELECT * FROM #db_growth



44. Script to identify what is going on currently in our SQL server.sql


SELECT r.session_id, 
s.program_name, 
s.login_name, 
r.start_time, 
r.status, 
r.command, 
Object_name(sqltxt.objectid, sqltxt.dbid) AS ObjectName, 
Substring(sqltxt.text, ( r.statement_start_offset / 2 ) + 1, ( ( 
CASE r.statement_end_offset 
WHEN -1 THEN 
datalength(sqltxt.text) 
ELSE r.statement_end_offset 
END 
- r.statement_start_offset ) / 2 ) + 1) AS active_statement,
r.percent_complete, 
Db_name(r.database_id) AS DatabaseName, 
r.blocking_session_id, 
r.wait_time, 
r.wait_type, 
r.wait_resource, 
r.open_transaction_count, 
r.cpu_time,-- in milli sec 
r.reads, 
r.writes, 
r.logical_reads, 
r.row_count, 
r.prev_error, 
r.granted_query_memory, 
Cast(sqlplan.query_plan AS XML) AS QueryPlan, 
CASE r.transaction_isolation_level 
WHEN 0 THEN 'Unspecified' 
WHEN 1 THEN 'ReadUncomitted' 
WHEN 2 THEN 'ReadCommitted' 
WHEN 3 THEN 'Repeatable' 
WHEN 4 THEN 'Serializable' 
WHEN 5 THEN 'Snapshot' 
END AS Issolation_Level, 
r.sql_handle, 
r.plan_handle 
FROM sys.dm_exec_requests r WITH (nolock) 
INNER JOIN sys.dm_exec_sessions s WITH (nolock) 
ON r.session_id = s.session_id 
CROSS apply sys.Dm_exec_sql_text(r.sql_handle) sqltxt 
CROSS apply 
sys.Dm_exec_text_query_plan(r.plan_handle, r.statement_start_offset, r.statement_end_offset) sqlplan
WHERE r.status <> 'background' 
ORDER BY r.session_id 
go 


45. Check the major product version to see if it is SQL Server 2019 CTP 2 or greater

IF NOT EXISTS (SELECT * WHERE CONVERT(varchar(128), SERVERPROPERTY('ProductVersion')) LIKE '15%')
BEGINDECLARE @ProductVersion varchar(128) = CONVERT(varchar(128), SERVERPROPERTY('ProductVersion'));
	RAISERROR ('Script does not match the ProductVersion [%s] of this instance. Many of these queries may not work on this version.' , 18 , 16 , @ProductVersion); END

-- SQL and OS Version information for current instance  (Query 1) (Version Info)
SELECT @@SERVERNAME AS [Server Name], @@VERSION AS [SQL Server and OS Version Info];



-- Instance level queries *******************************

-- SQL and OS Version information for current instance  (Query 1) (Version Info)
SELECT @@SERVERNAME AS [Server Name], @@VERSION AS [SQL Server and OS Version Info];



46. How to change compatibility level 

USE MASTER
GO

DECLARE @name varchar (255)
DECLARE @SQL varchar (max)

DECLARE cur_X cursor for
SELECT name from sys.databases
where database_ID >4

open cur_x

FETCH NEXT FROM CUR_X INTO @name 

WHILE @@FETCH_STATUS=0


BEGIN 

SET @SQL= 'ALTER DATABASE' + @name + 'SET compatibility_level=150'

print @SQL 

FETCH NEXT FROM cur_x INTO @name

END 

CLOSE Cur_x 

DEALLOCATE cur_X

 GO 

-------------------------------------------------------------------------------------------------------
 ---------------
 ALTER DATABASE [NORTHWNDS] SET compatibility level=150
ALTER DATABASET [SQL2012SET] SET compatibility level=150

--------------

USE MASTER

SELECT * FROM 
SYS.DATABASES

SELECT name, compatibility level from SYS. DATABASES

SELECT name, compatibility level from SYS. DATABASES

ALTER DATABASE[ MASTER ] SET compatibility level=150


SELECT name from SYS. DATABASES
where databased>2

---------------

print (@SQL)
EXEC (@SQL)






----------> No of data fille MDF LDF sp_helpdb 'NORTHWND'


47. Configure TDE

----
Select name, is_encrypted from sys.databases

--Step1:Take the full database backup of Dba
BACKUP DATABASE [dba] TO  DISK = N'D:\sqlbackups\dba.bak' WITH NOFORMAT, NOINIT,  
NAME = N'dba-Full Database Backup', SKIP, NOREWIND, NOUNLOAD,  STATS = 10
GO

--Step2:Create database master key
USE master;
Go
CREATE MASTER KEY 
ENCRYPTION BY PASSWORD = 'HYd@123';
GO

--Step3:Create certificate
USE master;
GO 
CREATE CERTIFICATE TDE_Certificate
       WITH SUBJECT='Certificate for TDE';
GO

--Step4:Create database encryption key
USE dba
GO
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Certificate;  

--Step5:Back up the certificate and the private key associated with the certificate
USE master;
GO
BACKUP CERTIFICATE [TDE_Certificate]
TO FILE = 'D:\harsha\TDE_Certificate_For_dbadatabase.cer'
WITH PRIVATE KEY (file='D:\harsha\TDE_dba_private_CertKey.pvk',
ENCRYPTION BY PASSWORD='HYd@123');

--Step6:Turn on encryption on database
ALTER DATABASE dba
SET ENCRYPTION ON

--Step8:Check encryption enabled

Select name, is_encrypted from sys.databases
Select * from sys.certificates
---------------------------------------------------------------

Backup mantainnance plan 

	1. Check SQL Server Instance Uptime
--In SQL Server 2008 or later you can also base your query on:
--SELECT sqlserver_start_time FROM sys.dm_os_sys_info
SELECT
create_date AS LastStartTime,
DATEDIFF(HOUR,create_date,GETDATE()) AS HoursSinceLastRestart,

CASE WHEN DATEDIFF(HOUR,create_date,GETDATE())>=24 THEN '100%' ELSE CON-
CAT(CAST(CAST(ROUND((DATEDIFF(HOUR,create_date,GETDATE())/0.24),2) AS NUMERIC(12,2)) AS

VARCHAR(10)),'%') END AS UpTimePercentageInLast24Hours
FROM sys.databases
WHERE

2. Check Last Full, Differential and Log Backup Times

SELECT d.[Name] AS DatabaseName,
LastFullBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
bs.[database_name]=d.[name] AND bs.[type]='D'),
LastDiffBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
bs.[database_name]=d.[name] AND bs.[type]='I'),
LastLogBackUpTime=(SELECT MAX(bs.backup_finish_date) FROM msdb.dbo.backupset bs WHERE
bs.[database_name]=d.[name] AND bs.[type]='L')
FROM sys.databases d
ORDER BY


3. Check Failed SQL Agent Jobs in the Last 24 Hours

SELECT sj.[name] AS "Failed Job Name", sh.run_date,sh.[message]
FROM msdb.dbo.sysjobhistory sh INNER JOIN msdb.dbo.sysjobs sj ON sj.job_id =
sh.job_id
WHERE sh.run_status=0 AND DATEDIFF(dd,cast (cast (sh.run_date AS VARCHAR(20)) AS
DATE),GETDATE())=0
GROUP BY sj.[name],sh.run_date,sh.message
GO


4. Check Current Log for Failed Logins

EXEC sp_readerrorlog 0, 1, 'Login failed';


5. Check for Blocked Processes

USE [master];
GO
SELECT r.session_id, r.blocking_session_id,
r.wait_type, r.wait_time,
r.wait_resource, r.transaction_isolation_level,
r.[lock_timeout], st.[text] AS BlockedSQLStatement
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st
WHERE blocking_session_id <> 0;
GO



6. Check Top N Queries Based on Average CPU Time

USE [Database_Name]
GO
SELECT TOP 10 qStatsFinal.query_hash AS "Query Plan Handle" ,
SUM(qStatsFinal.total_worker_time) / SUM(qStatsFinal.execution_count) AS
"Average CPU Time (ms)" ,

SUM(qStatsFinal.execution_count) AS "Execution Count",
SUM(qStatsFinal.total_logical_reads) AS "Total Logical Reads",
SUM(qStatsFinal.total_logical_writes) AS "Total Logical Writes",
SUM(qStatsFinal.total_physical_reads) AS "Total Physical Reads",

MAX(qStatsFinal.statement_text) AS "Statement Text"
FROM ( SELECT qStats.* ,
SUBSTRING(
sqlText.text ,
( qStats.statement_start_offset / 2 ) + 1,
(( CASE statement_end_offset
WHEN -1 THEN DATALENGTH(sqlText.text)
ELSE qStats.statement_end_offset
END - qStats.statement_start_offset ) / 2 ) + 1) AS
statement_text
FROM sys.dm_exec_query_stats AS qStats
CROSS APPLY sys.dm_exec_sql_text(qStats.sql_handle) AS sqlText
) AS qStatsFinal
GROUP BY qStatsFinal.query_hash
ORDER BY 2 DESC;
GO


7. Check Drive Volumes Space

SELECT
volume_mount_point AS "Drive Letter/Mount Point",
CAST(ROUND(MAX(total_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Disk Size (GB)",
CAST(ROUND(MAX(available_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Free Space (GB)"
FROM sys.master_files AS f
CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id)
GROUP BY volume_mount_point
ORDER BY 3 DESC
GO


8. Check Index Fragmentation

USE [DatabaseName];
GO
SELECT DB_NAME() AS DatabaseName ,
s.[object_id] ,
o.name AS ObjectName ,
index_type_desc ,
s.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(DB_NAME()), NULL, NULL, NULL, NULL) s
JOIN sys.objects o ON o.object_id = s.object_id
WHERE avg_fragmentation_in_percent <> 0
ORDER BY avg_fragmentation_in_percent DESC;
GO


9. Patch Management

You need keep your SQL Server instances up to date with the latest service packs and cumulative updates (CUs), after of
course you first test them in a Test Environment.
Service packs fix possible bugs, introduce new functionality and enhance security.
To see the latest updates for all SQL Server versions, you can visit the Update Center for Microsoft SQL Server
Also, for checking out what the latest service pack is for any SQL Server version, you can check out this free service by
SQLNetHub.Latest updates for SQL Server - SQL Server | Microsoft Docs


10. Self-Education and Improvement

Technology is evolving rapidly. So as database technologies. We live in a dynamic, technological environment. A good
DBA must get self-educated on a daily basis, not only about security patches and service packs, but also for many other
things that have to do with technology.
For example, you need to know about the latest SQL Server tools, related technologies, features, and so on.

This is the purpose of SQLNetHub: to help you stay up to date with anything that has to with SQL Server! From high-
quality articles on SQL Server, data access and .NET, to useful SQL Server tools and eBooks, SQLNetHub can help you with
your quest to comprehensive SQL Server knowledge!


------------------------------------------------------------------------------------------------------


------------- Step 1: Only show requests with active queries except for this one

SELECT er.session_id, er.command, er.status, er.wait_type, er.cpu_time, er.logical_reads, eqsx.query_plan, t.text
FROM sys.dm_exec_requests er
CROSS APPLY sys.dm_exec_query_statistics_xml(er.session_id) eqsx
CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) t
WHERE er.session_id <> @@SPID
GO
 
-- Step 2: What does the plan profile look like for the active query

-- Run the next query several time and see the progression of the row_count for
-- the Nested Loops and Table Spool operators

SELECT session_id, physical_operator_name, node_id, thread_id, row_count, estimate_row_count
FROM sys.dm_exec_query_profiles
WHERE session_id <> @@SPID
ORDER BY session_id, node_id DESC
GO

/*	
	Notice the huge estimate_row_count for the Nested Loops and Table Spool operators. 
	Notice the row_count (number of rows currently processed) is not even close to the estiamte.
	
	Is the estimate inaccurate? It is a possibility but, if it is right, this query is
	far from completing. 
*/

/* When lightweight query profiling is on by default in SQL Server 2019,
	row_count is the only statistics captured. Capturing statistics for CPU and I/O 
	can e expensive but you can sill can capture them with standard profiling.*/

-- Step 3: Go back and look at the plan and query text for a clue
SELECT er.session_id, er.command, er.status, er.wait_type, er.cpu_time, er.logical_reads, eqsx.query_plan, t.text
FROM sys.dm_exec_requests er
CROSS APPLY sys.dm_exec_query_statistics_xml(er.session_id) eqsx
CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) t
WHERE er.session_id <> @@SPID
GO

/* 
	Click on the XML plan. A new window will open and show the Plan in a Graphical way. 
	Get the value for the text column. This is the query being executed. Rewie the query 

	Notice the join clause
	INNER JOIN Sales.InvoiceLines sil
	ON si.InvoiceID = si.InvoiceID

	It is a self join. So the cause of the problem can be a simple typo
	and the query msut be modified to use the follwoing JOIN clause

	INNER JOIN Sales.InvoiceLines sil
	ON si.InvoiceID = sil.InvoiceID

 	
	The query take foreer to complete so it can be killed and fixed

	Kill the session for the mysmartquery.sql
*/


------------------------------------------------------

USE AdventureWorksPTO
GO

-- SETUP
DECLARE @UpperBound INT = 500000;

;WITH cteN(Number) AS
(
SELECT ROW_NUMBER() OVER (ORDER BY s1.[object_id]) - 1
FROM sys.all_columns AS s1
CROSS JOIN sys.all_columns AS s2
)
SELECT [Number], REPLICATE('0', 2000) as IntCounter INTO dbo.Numbers
FROM cteN WHERE [Number] <= @UpperBound;

CREATE UNIQUE CLUSTERED INDEX CIX_Number ON dbo.Numbers([Number])
WITH
(
FILLFACTOR = 100, -- in the event server default has been changed
DATA_COMPRESSION = ROW -- if Enterprise & table large enough to matter
);



-- Display the current waits

SELECT wait_type, wait_resource, count(*) AS cnt 
FROM sys.dm_exec_requests 
WHERE command = 'update'
GROUP BY wait_type, wait_resource



-- if time allows, 
-- show the remaining queries and the impact of the workload on the MI resource governance

-- Show all open transactions
SELECT * FROM SYS.dm_tran_database_transactions

-- Display the log bytes currently used in MB
SELECT SUM(database_transaction_log_bytes_used)/(1024*1024) as MB 
FROM SYS.dm_tran_database_transactions 

-- After starting the PowerShell workload, 
-- display the current log write impact in the column avg_log_write_percent
SELECT * FROM sys.dm_db_resource_stats

-- Show all connections coming from the OSTRESS workload
SELECT * FROM sys.dm_exec_connections

-- IO consumption
SELECT * FROM sys.dm_io_virtual_file_stats(null, null)
WHERE database_id = DB_ID('AdventureWorksPTO')
AND file_id = 2 -- emphasize 

-- IO possible issues
SELECT * FROM sys.dm_io_pending_io_requests

-- Edit the query to show the latest snapshot time. Display the resource usage deltas and stalls
SELECT * FROM 
sys.dm_resource_governor_workload_groups_history_ex
WHERE snapshot_time > '2020-11-25 17:38:38.283' AND pool_id = 2


All queries

SCOM SQL queries - Kevin Holman's Blog

=> SQL Server – Get all Login Accounts Using T-SQL Query – SQL Logins, Windows Logins, Windows Groups | DataGinger.com


	1. Deprecated logical disk free space  
	
	Check Drive Volumes Space
	
	SELECT
	volume_mount_point AS "Drive Letter/Mount Point",
	CAST(ROUND(MAX(total_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Disk Size (GB)",
	CAST(ROUND(MAX(available_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Free Space (GB)"
	FROM sys.master_files AS f
	CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id)
	GROUP BY volume_mount_point
	ORDER BY 3 DESC
	GO
	
	=>  Identify Disk Bottleneck in SQL Server using Perfmon Disk Counters (techyaz.com)

	2. Available Megabyte of Memory  



Script: Captures System Memory Usage

-- Works On: 2008, 2008 R2, 2012, 2014, 2016
/*********************************************************************/
select
      total_physical_memory_kb/1024 AS total_physical_memory_mb,
      available_physical_memory_kb/1024 AS available_physical_memory_mb,
      total_page_file_kb/1024 AS total_page_file_mb,
      available_page_file_kb/1024 AS available_page_file_mb,
      100 - (100 * CAST(available_physical_memory_kb AS DECIMAL(18,3))/CAST(total_physical_memory_kb AS DECIMAL(18,3))) 
      AS 'Percentage_Used',
      system_memory_state_desc
from  sys.dm_os_sys_memory;



 Script: SQL Server Process Memory Usage
-- Works On: 2008, 2008 R2, 2012, 2014, 2016
/**************************************************************/
select
      physical_memory_in_use_kb/1048576.0 AS 'physical_memory_in_use (GB)',
      locked_page_allocations_kb/1048576.0 AS 'locked_page_allocations (GB)',
      virtual_address_space_committed_kb/1048576.0 AS 'virtual_address_space_committed (GB)',
      available_commit_limit_kb/1048576.0 AS 'available_commit_limit (GB)',
      page_fault_count as 'page_fault_count'
from  sys.dm_os_process_memory;


Memory or RAM usage check script
--
select
(physical_memory_in_use_kb/1024)Phy_Memory_usedby_Sqlserver_MB,
(locked_page_allocations_kb/1024 )Locked_pages_used_Sqlserver_MB,
(virtual_address_space_committed_kb/1024 )Total_Memory_UsedBySQLServer_MB,
process_physical_memory_low,
process_virtual_memory_low
from sys. dm_os_process_memory



SQL Query to check buffer catch hit ratio
--
SELECT counter_name as CounterName, (a.cntr_value * 1.0 / b.cntr_value) * 100.0 as BufferCacheHitRatio
FROM sys.dm_os_performance_counters  a JOIN  (SELECT cntr_value,OBJECT_NAME FROM sys.dm_os_performance_counters
WHERE counter_name = 'Buffer cache hit ratio base' AND OBJECT_NAME LIKE '%Buffer Manager%') b ON  a.OBJECT_NAME = b.OBJECT_NAME 
WHERE a.counter_name =
'Buffer cache hit ratio' AND a.OBJECT_NAME LIKE '%Buffer Manager%



 Check which is using more memory

Select * from sys.sysprocesses where cpu > 100000
select * from sys.sysprocesses where memusage > xxxxx value
---Check which is using more memory and cpu
---Take that session id and put it in DBCC INPUTBUFFER  (SESSION ID)


 SQL server Memory Dumps


/* SQL Server Memory Dumps */
SELECT @@SERVERNAME AS [Server Name], [filename] AS [File Name], creation_time AS [Creation Time], 
size_in_bytes AS [Size in Bytes], CURRENT_TIMESTAMP AS [Collection Time] 
FROM sys.dm_server_memory_dumps WITH (NOLOCK) OPTION (RECOMPILE);


SQL Server Ring Buffer Memory Warnings 


/* SQL Server Ring Buffer Memory Warnings */
WITH     RingBuffer
AS       (SELECT CAST (dorb.record AS XML) AS xRecord,
                 dorb.timestamp
          FROM   sys.dm_os_ring_buffers AS dorb
          WHERE  dorb.ring_buffer_type = 'RING_BUFFER_RESOURCE_MONITOR')
SELECT   @@SERVERNAME AS [Server Name],
	 xr.value('(ResourceMonitor/Notification)[1]', 'VARCHAR(75)') AS RmNotification,
         xr.value('(ResourceMonitor/IndicatorsProcess)[1]', 'tinyint') AS IndicatorsProcess,
         xr.value('(ResourceMonitor/IndicatorsSystem)[1]', 'tinyint') AS IndicatorsSystem,
         DATEADD (ss, (-1 * ((dosi.cpu_ticks / CONVERT (float, ( dosi.cpu_ticks / dosi.ms_ticks ))) - [timestamp])/1000), CURRENT_TIMESTAMP) AS RmDateTime,
         xr.value('(MemoryNode/TargetMemory)[1]', 'BIGINT') AS TargetMemory,
         xr.value('(MemoryNode/ReserveMemory)[1]', 'BIGINT') AS ReserveMemory,
         xr.value('(MemoryNode/CommittedMemory)[1]', 'BIGINT') AS CommitedMemory,
         xr.value('(MemoryNode/SharedMemory)[1]', 'BIGINT') AS SharedMemory,
         xr.value('(MemoryNode/PagesMemory)[1]', 'BIGINT') AS PagesMemory,
         xr.value('(MemoryRecord/MemoryUtilization)[1]', 'BIGINT') AS MemoryUtilization,
         xr.value('(MemoryRecord/TotalPhysicalMemory)[1]', 'BIGINT') AS TotalPhysicalMemory,
         xr.value('(MemoryRecord/AvailablePhysicalMemory)[1]', 'BIGINT') AS AvailablePhysicalMemory,
         xr.value('(MemoryRecord/TotalPageFile)[1]', 'BIGINT') AS TotalPageFile,
         xr.value('(MemoryRecord/AvailablePageFile)[1]', 'BIGINT') AS AvailablePageFile,
         xr.value('(MemoryRecord/TotalVirtualAddressSpace)[1]', 'BIGINT') AS TotalVirtualAddressSpace,
         xr.value('(MemoryRecord/AvailableVirtualAddressSpace)[1]', 'BIGINT') AS AvailableVirtualAddressSpace,
         xr.value('(MemoryRecord/AvailableExtendedVirtualAddressSpace)[1]', 'BIGINT') AS AvailableExtendedVirtualAddressSpace,
    CURRENT_TIMESTAMP AS [Collection Time]
FROM     RingBuffer AS rb CROSS APPLY rb.xRecord.nodes ('Record') AS record(xr) CROSS JOIN sys.dm_os_sys_info AS dosi
ORDER BY RmDateTime DESC;



	3. Average Disk second per write (Logical Disk)

    Check Drive Volumes Space

SELECT
volume_mount_point AS "Drive Letter/Mount Point",
CAST(ROUND(MAX(total_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Disk Size (GB)",
CAST(ROUND(MAX(available_bytes)/1024.0/1024.0/1024.0,2) AS INT) AS "Free Space (GB)"
FROM sys.master_files AS f
CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id)
GROUP BY volume_mount_point
ORDER BY 3 DESC
GO




	4. Average Disk second per write (Physical Disk)

=> Physical Disk - Average Disk Seconds Per Write - Microsoft.Windows.Client.Win10.PhysicalDisk.AvgDiskSecPerWrite (UnitMonitor) (systemcenter.wiki)


	5. Average logical Disk second per read 

=> Logical Disk Average Disk Second Per Read - Rules - Public MPWiki (mpwikiapp-stage.azurewebsites.net)


	6. Average logical Disk second per transfer 



	7. Computer Browser service Health  



	8. Core window service Rollup






	9. CPU DPC Time Percentage 






Check CPU by Database

/* CPU by Database */
WITH DB_CPU_Stats
AS
(SELECT DatabaseID, DB_Name(DatabaseID) AS [DatabaseName], SUM(total_worker_time) AS [CPU_Time_Ms]
 FROM sys.dm_exec_query_stats AS qs
 CROSS APPLY (SELECT CONVERT(INT, value) AS [DatabaseID] 
              FROM sys.dm_exec_plan_attributes(qs.plan_handle)
              WHERE attribute = N'dbid') AS F_DB
 GROUP BY DatabaseID)
SELECT @@SERVERNAME AS [Server Name], ROW_NUMBER() OVER(ORDER BY [CPU_Time_Ms] DESC) AS [Row Number],
       DatabaseName AS [Database Name], [CPU_Time_Ms] AS [CPU Time MS], 
       CAST([CPU_Time_Ms] * 1.0 / SUM([CPU_Time_Ms]) OVER() * 100.0 AS DECIMAL(5, 2)) AS [CPU Percent], 
	  CURRENT_TIMESTAMP AS [Collection Time] 
FROM DB_CPU_Stats
WHERE DatabaseID <> 32767 -- ResourceDB
ORDER BY [Row Number] OPTION (RECOMPILE);


CPU Utilization History 

--

/* CPU Utilization History (last 144 minutes in one minute intervals) */
DECLARE @ts_now BIGINT = ( SELECT
							cpu_ticks / ( cpu_ticks / ms_ticks )
							FROM
							sys.dm_os_sys_info
							) ; 
SELECT TOP ( 144 )
	@@SERVERNAME AS [Server Name],
	SQLProcessUtilization AS [SQL Server Process CPU Utilization] ,
	SystemIdle AS [System Idle Process] ,
	100 - SystemIdle - SQLProcessUtilization AS [Other Process CPU Utilization] ,
	DATEADD(ms, -1 * ( @ts_now - [timestamp] ), CURRENT_TIMESTAMP) AS [Event Time],
	CURRENT_TIMESTAMP AS [Collection Time]
FROM
	( SELECT
		record.value('(./Record/@id)[1]', 'INT') AS record_id ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]',
						'INT') AS [SystemIdle] ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
						'INT') AS [SQLProcessUtilization] ,
		[timestamp]
		FROM
		( SELECT
			[timestamp] ,
			CONVERT(XML, record) AS [record]
			FROM
			sys.dm_os_ring_buffers
			WHERE
			ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
			AND record LIKE N'%<SystemHealth>%'
		) AS x
	) AS y
ORDER BY
	record_id DESC
OPTION
	(RECOMPILE);




  CPU Utilization History  (last 144 minutes in one minute intervals)

SET ANSI_NULLS, QUOTED_IDENTIFIER ON;
GO
DECLARE @Duration INT=240-------------You can put here Duration like 60/120
DECLARE @ts_now BIGINT = ( SELECT
cpu_ticks / ( cpu_ticks / ms_ticks )
FROM
sys.dm_os_sys_info
) ;
SELECT TOP ( @Duration )
@@SERVERNAME AS [Server Name],
SQLProcessUtilization AS [SQL Server Process CPU Utilization] ,
SystemIdle AS [System Idle Process] ,
100 - SystemIdle - SQLProcessUtilization AS [Other Process CPU Utilization] ,
DATEADD(ms, -1 * ( @ts_now - [timestamp] ), CURRENT_TIMESTAMP) AS [Event Time],
CURRENT_TIMESTAMP AS [Collection Time]
FROM
( SELECT
record.value('(./Record/@id)[1]', 'INT') AS record_id ,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]',
'INT') AS [SystemIdle] ,
record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
'INT') AS [SQLProcessUtilization] ,
[timestamp]
FROM
( SELECT
[timestamp] ,
CONVERT(XML, record) AS [record]
FROM
sys.dm_os_ring_buffers
WHERE
ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
AND record LIKE N'%<SystemHealth>%'
) AS x
) AS y
ORDER BY
record_id DESC
OPTION
(RECOMPILE);



  Processor name and Description 

/* Processor Description */
CREATE TABLE #RDXResults
(value VARCHAR(255), data VARCHAR(255));
INSERT INTO #RDXResults
EXEC xp_instance_regread 'HKEY_LOCAL_MACHINE','HARDWARE\DESCRIPTION\System\CentralProcessor\0','ProcessorNameString';
SELECT @@SERVERNAME AS [Server Name], value AS [Value Name], data AS [Processor Information], CURRENT_TIMESTAMP AS [Collection Time] FROM #RDXResults;
DROP TABLE #RDXResults;






	10. CPU Percentage interrupt time  


=> Finding SQL Server CPU problems with DMV queries - Longitude (heroix.com)

	11. CPU Percentage utilization   


CPU Utilization History 

--

/* CPU Utilization History (last 144 minutes in one minute intervals) */
DECLARE @ts_now BIGINT = ( SELECT
							cpu_ticks / ( cpu_ticks / ms_ticks )
							FROM
							sys.dm_os_sys_info
							) ; 
SELECT TOP ( 144 )
	@@SERVERNAME AS [Server Name],
	SQLProcessUtilization AS [SQL Server Process CPU Utilization] ,
	SystemIdle AS [System Idle Process] ,
	100 - SystemIdle - SQLProcessUtilization AS [Other Process CPU Utilization] ,
	DATEADD(ms, -1 * ( @ts_now - [timestamp] ), CURRENT_TIMESTAMP) AS [Event Time],
	CURRENT_TIMESTAMP AS [Collection Time]
FROM
	( SELECT
		record.value('(./Record/@id)[1]', 'INT') AS record_id ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]',
						'INT') AS [SystemIdle] ,
		record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]',
						'INT') AS [SQLProcessUtilization] ,
		[timestamp]
		FROM
		( SELECT
			[timestamp] ,
			CONVERT(XML, record) AS [record]
			FROM
			sys.dm_os_ring_buffers
			WHERE
			ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
			AND record LIKE N'%<SystemHealth>%'
		) AS x
	) AS y
ORDER BY
	record_id DESC
OPTION
	(RECOMPILE);




	12. Current Disk queue length (logical disk) 

=> Identify Disk Bottleneck in SQL Server using Perfmon Disk Counters (techyaz.com)


	13. Current Disk queue length (Physical disk) 

=> Disk Queue Length - an overview | ScienceDirect Topics



	14. DHCP client service health 

=> Troubleshoot problems on the DHCP client | Microsoft Learn


	15. DNS Clints  service health 

=> How to check DNS health? (windowstechno.com)

	16. File system error or corruption


-------------------------------------------------------------------- 


Performance tuning queries 

SELECT *
FROM sys.dm_exec_tasks;
-- relevant data: 
-- task_state --> running, suspended
-- pending_io_* --> I/O activity
-- scheduler_id -->processor info
-- session_id --> spid


------ Perfromance tuning 


SELECT *
FROM sys.dm_exec_requests;
-- relevant data: 
-- session_id --> spid
-- status --> background, running, runnable, suspended
-- sql_handle, offset --> query text
-- database_id --> database being accessed
-- wait_type, wait_time --> blocking information
-- open_transaction_count --> blocking others
-- cpu_time, total_elapsed_time, reads, writes --> telemetry


---------------


SELECT *
FROM sys.dm_exec_sessions;
-- relevant data: 
-- session_id --> spid
-- host_name, program_name --> client identity 
-- login_name, nt_user_name --> login identity
-- status --> activity
-- database_id --> database being accessed
-- open_transaction_count --> blocking identification


--- Get the list of all Windows Group Login Accounts only
 
SELECT name
FROM sys.server_principals 
WHERE TYPE = 'G'


----- Get the list of all Windows Login Accounts only

SELECT name
FROM sys.server_principals 
WHERE TYPE = 'U'



---- Get the list of all SQL Login Accounts only

SELECT name
FROM sys.server_principals 
WHERE TYPE = 'S'
and name not like '%##%'




-------- Get the list of all Login Accounts in a SQL Server

SELECT name AS Login_Name, type_desc AS Account_Type
FROM sys.server_principals 
WHERE TYPE IN ('U', 'S', 'G')
and name not like '%##%'
ORDER BY name, type_desc

 --- Get the list of all Windows Group Login Accounts only
 
SELECT name
FROM sys.server_principals 
WHERE TYPE = 'G'


----- Get the list of all Windows Login Accounts only

SELECT name
FROM sys.server_principals 
WHERE TYPE = 'U'



---- Get the list of all SQL Login Accounts only

SELECT name
FROM sys.server_principals 
WHERE TYPE = 'S'
and name not like '%##%'




-------- Get the list of all Login Accounts in a SQL Server

SELECT name AS Login_Name, type_desc AS Account_Type
FROM sys.server_principals 
WHERE TYPE IN ('U', 'S', 'G')
and name not like '%##%'
ORDER BY name, type_desc




-- The following update generates about 5MB of log content
-- Run in OSTRESS

UPDATE dbo.Numbers 
SET IntCounter = REPLICATE('a', 2000) 
WHERE [Number] >= @@spid * 1000
AND [Number] <= @@spid * 1000 + 900

--------------------------------------------


Azure 


CREATE EVENT SESSION [Test Scripter] ON DATABASE 
ADD EVENT sqlserver.error_reported(
    ACTION(sqlserver.client_app_name,sqlserver.database_id,sqlserver.query_hash,sqlserver.session_id)
    WHERE ([package0].[greater_than_uint64]([sqlserver].[database_id],(4)) AND [package0].[equal_boolean]([sqlserver].[is_system],(0)))),
ADD EVENT sqlserver.rpc_starting(SET collect_statement=(1)
    ACTION(package0.event_sequence,sqlserver.client_app_name,sqlserver.database_id)),
ADD EVENT sqlserver.sql_batch_starting(
    ACTION(package0.event_sequence,sqlserver.client_app_name,sqlserver.database_id))
ADD TARGET package0.ring_buffer
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=ON,STARTUP_STATE=OFF)
GO

---------------------------------------

SELECT DISTINCT( s.NAME )
FROM
  sys.database_event_session_events se
  JOIN sys.database_event_sessions s
    ON s.event_session_id = se.event_session_id
GO

select * from sys.database_event_session_events
select * from sys.database_event_sessions


SELECT
        o.object_type,
        p.name         AS [package_name],
        o.name         AS [db_object_name],
        o.description  AS [db_obj_description]
    FROM
                   sys.dm_xe_objects  AS o
        INNER JOIN sys.dm_xe_packages AS p  ON p.guid = o.package_guid
    WHERE
        o.object_type in
            (
            'action',  'event',  'target'
            )
    ORDER BY
        o.object_type,
        p.name,
        o.name;




-------------------------------


CREATE EVENT SESSION [Test Scripter] ON DATABASE 
ADD EVENT sqlserver.error_reported(
    ACTION(sqlserver.client_app_name,sqlserver.database_id,sqlserver.query_hash,sqlserver.session_id)
    WHERE (([package0].[greater_than_uint64]([sqlserver].[database_id],(4))) 
	AND ([package0].[equal_boolean]([sqlserver].[is_system],(0))))),
ADD EVENT sqlserver.rpc_starting(SET collect_statement=(1)
    ACTION(package0.event_sequence,sqlserver.client_app_name,sqlserver.database_id)),
ADD EVENT sqlserver.sql_batch_starting(
    ACTION(package0.event_sequence,sqlserver.client_app_name,sqlserver.database_id))
ADD TARGET package0.ring_buffer
WITH (MAX_MEMORY=4096 KB,EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,MAX_DISPATCH_LATENCY=30 SECONDS,MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE,TRACK_CAUSALITY=ON,STARTUP_STATE=OFF)
GO






---------------------------------------------------------------------------------------------------------------------------


Hi Philip,
 
As far as I am aware, the app team continue to report off the readable secondary - but they hinted that they had/we working on improvements
 
This is what we’ve got in play - some tidy-ups along the way in the script. We’ll try to run SQL scout if there is are-occurrence.
 
I’ve uploaded the extended event file to date. ~17:00 as a time seems to appear more frequently than might be expected.
 
 
-- drop the old, basic session we created after the inicdent
-- m2z-pr-pdwdb-01 and 02

DROP EVENT SESSION [INC3564351] ON SERVER
GO

-- node 2, readonly secondary
-- value is 0 on 17th July

sp_configure 'show advanced options', 1 ;
GO
RECONFIGURE ;
GO
sp_configure 'blocked process threshold'

-- let's set to 30
-- node 2

sp_configure 'show advanced options', 1 ;
GO
RECONFIGURE ;
GO
sp_configure 'blocked process threshold', 30 ;
GO
RECONFIGURE ;
GO

--Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
--Configuration option 'blocked process threshold (s)' changed from 0 to 30. Run the RECONFIGURE statement to install.

--Completion time: 2023-07-17T08:43:33.6900735-04:00


CREATE EVENT SESSION [TKT41717406] ON SERVER
ADD EVENT sqlserver.lock_redo_blocked(
ACTION(sqlserver.nt_username,sqlserver.session_id,sqlserver.sql_text)),
ADD EVENT sqlserver.lock_redo_unblocked(
ACTION(sqlserver.nt_username,sqlserver.session_id,sqlserver.sql_text))
ADD TARGET package0.event_file(SET filename=N'TKT41717406')
WITH (STARTUP_STATE=ON)
GO
 
 
--Node 2
	 

DROP EVENT SESSION [TKT41717406] ON SERVER

CREATE EVENT SESSION [TKT41717406] ON SERVER
ADD EVENT sqlserver.lock_redo_blocked, --( ACTION(sqlserver.nt_username,sqlserver.session_id,sqlserver.sql_text)),
ADD EVENT sqlserver.lock_redo_unblocked --( ACTION(sqlserver.nt_username,sqlserver.session_id,sqlserver.sql_text))
ADD TARGET package0.event_file(SET filename=N'TKT41717406')
WITH (STARTUP_STATE=ON)
GO
 
Thanks,
Peter Schofield
 

---------------------------------------------------------------------------------------------------------------------------

Searching for: SQL server migration path step by step

Generating answers for you…

There are different methods and tools for migrating SQL Server databases to a new version or platform. Depending on your specific scenario and requirements, you may need to follow different steps and use different tools. Here are some general steps and best practices for SQL Server migration:

• Discover your SQL Server instances and databases and collect information about their configuration, performance, applications, and dependencies. You can use tools like MAP Toolkit1 or Azure Migrate2 to help you with this step.

• Assess your SQL Server databases for compatibility and feature parity with the target version or platform. You can use tools like Data Migration Assistant (DMA)13 or Azure Migrate: Database Assessment2 to help you with this step. These tools can identify potential issues or blockers, and provide recommendations and guidance for resolving them.

• Migrate your SQL Server databases to the target version or platform using the appropriate method and tool. Some common methods and tools are:

• Backup and restore: This method involves taking a full backup of the source database and restoring it to the target server. This method is simple and reliable, but it may require downtime and it may not work for cross-platform migrations. You can use tools like SQL Server Management Studio (SSMS) or Azure Data Studio to perform backup and restore operations.

• Detach and attach: This method involves detaching the source database from the source server, copying the database files to the target server, and attaching the database to the target server. This method is similar to backup and restore, but it may be faster and more efficient for large databases. However, it also requires downtime and it may not work for cross-platform migrations. You can use tools like SSMS or Azure Data Studio to perform detach and attach operations.

• Log shipping: This method involves taking a full backup of the source database, restoring it to the target server, and then applying transaction log backups periodically to keep the target database synchronized with the source database. This method allows minimal downtime and works for cross-platform migrations, but it may require additional configuration and monitoring. You can use tools like SSMS or SQL Server Agent to configure log shipping.

• Database mirroring: This method involves creating a mirror database on the target server that is an exact copy of the source database, and then synchronizing the changes between them using a witness server. This method allows minimal downtime and works for cross-platform migrations, but it may require additional hardware and network resources. You can use tools like SSMS or Transact-SQL to configure database mirroring.

• Always On availability groups: This method involves creating an availability group that contains replicas of the source database on different servers, and then performing a planned manual failover to switch the primary replica to the target server. This method allows minimal downtime and works for cross-platform migrations, but it may require additional licensing and configuration. You can use tools like SSMS, Transact-SQL, or PowerShell to configure Always On availability groups.

• Transactional replication: This method involves creating a publication on the source database that contains articles (objects) to be replicated, creating a subscription on the target database that receives the replicated data, and then using a distribution agent to apply the changes from the publisher to the subscriber. This method allows minimal downtime and works for cross-platform migrations, but it may require additional configuration and maintenance. You can use tools like SSMS, Replication Monitor, or Transact-SQL to configure transactional replication.

• SQL Server Integration Services (SSIS): This method involves creating an SSIS package that extracts data from the source database, transforms it if needed, and loads it into the target database. This method allows flexibility and customization, but it may require more development time and testing. You can use tools like SSIS Designer, SSIS Catalog, or dtexec to create and execute SSIS packages.

• Azure Database Migration Service (DMS): This method involves using a fully managed service that streamlines the migration of multiple databases from different sources to Azure data platforms with minimal downtime. This method supports online, offline, and hybrid migrations, but it may require additional configuration and validation. You can use tools like Azure portal, PowerShell, or REST API to create and monitor DMS activities.

• Test your migrated SQL Server databases for functionality, performance, security, and compatibility with your applications. You can

From <https://www.bing.com/search?q=Bing%2BAI&FORM=ANWSB6&PC=U531&showconv=1> 





Main problem SQL server migration





SQL Server migration can be a complex and challenging process, depending on the source and target versions, platforms, and environments. Some of the main problems that can occur during SQL Server migration are:

• Network connectivity issues: If the source and target SQL Server instances are not able to communicate with each other or with the migration tools, the migration process may fail or be interrupted. You should check the network connectivity for the SQL Server computer, and make sure that the Azure firewall or any other firewall is not blocking your SQL Server from connecting to the remote endpoint1.

• Compatibility and feature parity issues: If the source and target SQL Server instances have different versions or editions, some features or functionalities may not be supported or may behave differently on the target instance. You should assess your SQL Server databases for compatibility and feature parity with the target version or platform, using tools like Data Migration Assistant (DMA) or Azure Migrate: Database Assessment. These tools can identify potential issues or blockers, and provide recommendations and guidance for resolving them23.

• Data type issues: If the source and target SQL Server instances have different data types or collations, some data may not be migrated correctly or may cause errors. For example, online migrations of SQL_variant data types are not supported from SQL Server to Azure SQL Managed Instance4. You should check the data types and collations of your source and target SQL Server instances, and make sure they are compatible or convertible.

• Index and primary key issues: If the source SQL Server tables do not have indexes or primary keys, the migration process may be slower or may not work at all. SQL Server requires all tables to have at least one index and requires a linked table to have a primary key if the table can be updated5. You should add indexes and primary keys to your source SQL Server tables before migration, and make sure they are consistent with the target SQL Server tables.

• Application dependency issues: If your applications depend on certain features or functionalities of your source SQL Server instance, they may not work properly after migration to the target SQL Server instance. For example, if your applications use hard-coded connection strings or server names, they may not be able to connect to the new SQL Server instance. You should identify and update any application dependencies before migration, and test your applications for functionality, performance, security, and compatibility after migration.

From <https://www.bing.com/search?q=Bing%2BAI&FORM=ANWSB6&PC=U531&showconv=1> 




How to solve Compatibility issues during SQL Server migration



Compatibility issues during migration can occur when the source and target SQL Server instances have different versions, editions, features, or data types that are not supported or behave differently on the target instance. To solve compatibility issues during migration, you should:
• Assess your SQL Server databases for compatibility and feature parity with the target version or platform, using tools like Data Migration Assistant (DMA) or Azure Migrate: Database Assessment. These tools can identify potential issues or blockers, and provide recommendations and guidance for resolving them12.
• Transform your data into a compatible format, if needed, before migrating it. This can be done by breaking it down into smaller chunks, dividing it into various parts, and developing code3. You can also use tools like SQL Server Integration Services (SSIS) or Azure Database Migration Service (DMS) to extract, transform, and load your data.
• Test your migrated SQL Server databases for functionality, performance, security, and compatibility with your applications, after migration. You can use tools like Database Experimentation Assistant (DEA) or Azure Data Studio to compare the behavior and performance of your source and target SQL Server instances.

From <https://www.bing.com/search?q=Bing%2BAI&FORM=ANWSB6&PC=U531&showconv=1> 



Show me how to use DMA to assess my databases.


Data Migration Assistant (DMA) is a tool that helps you assess your SQL Server databases for compatibility and feature parity with the target version or platform. To use DMA to assess your databases, you can follow these steps:

• Download and install DMA from the Microsoft Download Center, and then run the DataMigrationAssistant.msi file.

• Create an assessment project by selecting the New (+) icon, and then selecting the Assessment project type. Set the source and target server type, and then click Create.

• Choose assessment options by selecting the target SQL Server version to which you plan to migrate, and selecting the report type. You can choose one or both of the following assessment report types:

• Check database compatibility: This report type detects compatibility issues that block migrating your SQL Server databases to the target platform, and provides recommendations to help you address those issues.

• Check feature parity: This report type detects partially supported or unsupported features that are currently in use on the source SQL Server instance, and provides recommendations, alternative approaches, and mitigating steps so that you can incorporate them into your migration projects.

• Add databases and extended events trace to assess by selecting Add Sources to open the connection flyout menu. Enter the SQL server instance name, choose the Authentication type, set the correct connection properties, and then select Connect. Select the databases to assess, and then select Add. If you have any ad hoc or dynamic SQL queries or any DML statements initiated through the application data layer, then enter the path to the folder in which you placed all the extended events session files that you collected to capture the workload on the source SQL Server.

• View results by selecting Start Assessment. DMA analyzes your source SQL Server instance and databases, and generates an assessment report that shows the compatibility issues and feature parity gaps for each database. You can view the details of each issue or gap, and see the recommendations and guidance provided by DMA. You can also export or save your assessment results for further analysis or reporting1.
For more information on how to use DMA to assess your databases, you can refer to this article: Perform a SQL Server migration assessment with Data Migration Assistant1

From <https://www.bing.com/search?q=Bing%2BAI&FORM=ANWSB6&PC=U531&showconv=1> 




---------- How to Script out  generate database users permission in Azure managed instance BY Victor ?

WITH perms_cte as
(select USER_NAME(p.grantee_principal_id) AS principal_name,
dp.principal_id,
dp.type_desc AS principal_type_desc,
p.class_desc,
OBJECT_NAME(p.major_id) AS object_name,
p.permission_name,
p.state_desc AS permission_state_desc
from sys.database_permissions p
inner JOIN sys.database_principals dp
on p.grantee_principal_id = dp.principal_id
)
SELECT p.principal_name, p.principal_type_desc, p.class_desc, p.[object_name], p.permission_name, p.permission_state_desc, cast(NULL as sysname) as role_name
FROM perms_cte p
WHERE principal_type_desc <> 'DATABASE_ROLE'
UNION
SELECT rm.member_principal_name, rm.principal_type_desc, p.class_desc, p.object_name, p.permission_name, p.permission_state_desc,rm.role_name
FROM perms_cte p
right outer JOIN (
select role_principal_id, dp.type_desc as principal_type_desc, member_principal_id,user_name(member_principal_id) as member_principal_name,user_name(role_principal_id) as role_name--,*
from sys.database_role_members rm
INNER JOIN sys.database_principals dp
ON rm.member_principal_id = dp.principal_id
) rm
ON rm.role_principal_id = p.principal_id
order by principal_name

If your boss asks you to restore the database until 2:25, and you have backups and transaction logs up to that point, you can follow these steps:

1. Full Backup Restoration: 
    - Start by restoring the latest full backup that was taken before 2:25. Make sure to restore with the `NORECOVERY` option. This leaves the database in a state where subsequent transaction log backups can be applied.

2. Differential Backup (if applicable):
    - If there's a differential backup available after the full backup and before 2:25, restore it next. Use the `NORECOVERY` option to keep the database in a restorable state.

3. Transaction Log Restoration:
    - Apply all transaction log backups sequentially that were taken after the full (or differential) backup until 2:25. 
    Use the `STOPAT` command to specify the exact point in time (2:25) for the restore.
    - For the last transaction log restore, use the `RECOVERY` option to bring the database online.

Here's a simple example using T-SQL:
-- Restore full backup
RESTORE DATABASE YourDatabase 
FROM DISK = 'Path_to_full_backup.bak' 
WITH NORECOVERY;

-- Restore differential backup (if applicable)
RESTORE DATABASE YourDatabase 
FROM DISK = 'Path_to_differential_backup.bak' 
WITH NORECOVERY;

-- Restore transaction log backups up to 2:25
RESTORE LOG YourDatabase 
FROM DISK = 'Path_to_first_log_backup.trn' 
WITH NORECOVERY;

-- Continue restoring any other transaction logs taken before 2:25


-- For the final log restore, use the STOPAT command and RECOVERY option
RESTORE LOG YourDatabase 
FROM DISK = 'Path_to_last_log_backup_before_2_25.trn' 
WITH RECOVERY, STOPAT = 'YYYY-MM-DD 14:25:00';


Make sure to replace `YourDatabase`, and the paths with the appropriate names/paths for your environment. Always remember to test the restore process in a separate environment first, if possible, to ensure data integrity and correctness.

Restoring the last transactional log until 2:25 


-- Restore full backup

RESTORE DATABASE YourDatabase 
FROM DISK = 'Path_to_full_backup.bak' 
WITH NORECOVERY;

-- Restore differential backup (if applicable)
RESTORE DATABASE YourDatabase 
FROM DISK = 'Path_to_differential_backup.bak' 
WITH NORECOVERY;

-- Restore transaction log backups up to 2:25
RESTORE LOG YourDatabase 
FROM DISK = 'Path_to_first_log_backup.trn' 
WITH NORECOVERY;

-- Continue restoring any other transaction logs taken before 2:25


-- For the final log restore, use the STOPAT command and RECOVERY option
RESTORE LOG YourDatabase 
FROM DISK = 'Path_to_last_log_backup_before_2_25.trn' 
WITH RECOVERY, STOPAT = 'YYYY-MM-DD 14:25:00';





