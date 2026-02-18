
Top 50 SQL Server DMV Scripts (DBA Daily + Triage)

 CPU & Query Performance (1–12) 
1) Top queries by CPU (total worker time)


SELECT TOP (50)
    qs.total_worker_time/1000.0 AS total_cpu_ms,
    qs.execution_count,
    (qs.total_worker_time/NULLIF(qs.execution_count,0))/1000.0 AS avg_cpu_ms,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC
OPTION (RECOMPILE);


2) Top queries by average CPU


SELECT TOP (50)
    (qs.total_worker_time/NULLIF(qs.execution_count,0))/1000.0 AS avg_cpu_ms,
    qs.execution_count,
    qs.total_worker_time/1000.0 AS total_cpu_ms,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY (qs.total_worker_time/NULLIF(qs.execution_count,0)) DESC
OPTION (RECOMPILE);


3) Top queries by logical reads (total)


SELECT TOP (50)
    (qs.total_logical_reads*8/1024.0) AS total_logical_reads_mb,
    qs.execution_count,
    (qs.total_logical_reads/NULLIF(qs.execution_count,0))*8/1024.0 AS avg_logical_reads_mb,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_logical_reads DESC
OPTION (RECOMPILE);


4) Top queries by average logical reads


SELECT TOP (50)
    (qs.total_logical_reads/NULLIF(qs.execution_count,0))*8/1024.0 AS avg_logical_reads_mb,
    qs.execution_count,
    (qs.total_logical_reads*8/1024.0) AS total_logical_reads_mb,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY (qs.total_logical_reads/NULLIF(qs.execution_count,0)) DESC
OPTION (RECOMPILE);


5) Top queries by elapsed time (total)


SELECT TOP (50)
    qs.total_elapsed_time/1000.0 AS total_elapsed_ms,
    qs.execution_count,
    (qs.total_elapsed_time/NULLIF(qs.execution_count,0))/1000.0 AS avg_elapsed_ms,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_elapsed_time DESC
OPTION (RECOMPILE);


6) Top queries by average elapsed time


SELECT TOP (50)
    (qs.total_elapsed_time/NULLIF(qs.execution_count,0))/1000.0 AS avg_elapsed_ms,
    qs.execution_count,
    qs.total_elapsed_time/1000.0 AS total_elapsed_ms,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY (qs.total_elapsed_time/NULLIF(qs.execution_count,0)) DESC
OPTION (RECOMPILE);


7) CPU by database


WITH DB_CPU AS
(
  SELECT
    CONVERT(int, pa.value) AS database_id,
    SUM(qs.total_worker_time) AS cpu_time
  FROM sys.dm_exec_query_stats qs
  CROSS APPLY sys.dm_exec_plan_attributes(qs.plan_handle) pa
  WHERE pa.attribute = N'dbid'
  GROUP BY CONVERT(int, pa.value)
)
SELECT TOP (50)
  DB_NAME(database_id) AS database_name,
  cpu_time/1000.0 AS total_cpu_ms,
  cpu_time * 1.0 / SUM(cpu_time) OVER() * 100.0 AS cpu_percent
FROM DB_CPU
WHERE database_id <> 32767
ORDER BY cpu_time DESC
OPTION (RECOMPILE);


8) Most frequently executed queries


SELECT TOP (50)
    qs.execution_count,
    qs.total_worker_time/1000.0 AS total_cpu_ms,
    (qs.total_worker_time/NULLIF(qs.execution_count,0))/1000.0 AS avg_cpu_ms,
    DB_NAME(st.dbid) AS database_name,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.execution_count DESC
OPTION (RECOMPILE);


9) Top cached plans by memory grant / cache size


SELECT TOP (50)
  cp.size_in_bytes/1024.0/1024.0 AS plan_cache_mb,
  cp.usecounts,
  st.text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
ORDER BY cp.size_in_bytes DESC
OPTION (RECOMPILE);


10) Ad hoc “single-use” plan cache bloat


SELECT
  COUNT(*) AS single_use_plans,
  SUM(size_in_bytes)/1024.0/1024.0 AS total_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1
  AND objtype = 'Adhoc'
OPTION (RECOMPILE);


11) Current memory grants (who’s waiting/granted)


SELECT TOP (50)
  mg.session_id,
  mg.requested_memory_kb/1024.0 AS requested_mb,
  mg.granted_memory_kb/1024.0 AS granted_mb,
  mg.wait_time_ms,
  mg.request_time,
  mg.grant_time,
  st.text
FROM sys.dm_exec_query_memory_grants mg
OUTER APPLY sys.dm_exec_sql_text(mg.sql_handle) st
ORDER BY mg.requested_memory_kb DESC
OPTION (RECOMPILE);


12) Query Store (if enabled): top regressed queries (2016+)


SELECT TOP (50)
  q.query_id,
  rs.avg_duration,
  rs.avg_cpu_time,
  rs.avg_logical_io_reads,
  qt.query_sql_text
FROM sys.query_store_query q
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
ORDER BY rs.avg_cpu_time DESC;


 Waits & Bottlenecks (13–22)

13) Top waits since restart (filtered)


SELECT TOP (50)
  wait_type,
  wait_time_ms/1000.0 AS wait_s,
  100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS pct,
  signal_wait_time_ms/1000.0 AS signal_wait_s
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE 'SLEEP%'
  AND wait_type NOT IN ('BROKER_TASK_STOP','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH','CLR_AUTO_EVENT','CLR_MANUAL_EVENT')
ORDER BY wait_time_ms DESC
OPTION (RECOMPILE);


14) Signal vs resource waits (CPU pressure indicator)


SELECT
  SUM(signal_wait_time_ms) AS total_signal_wait_ms,
  SUM(wait_time_ms - signal_wait_time_ms) AS total_resource_wait_ms
FROM sys.dm_os_wait_stats
OPTION (RECOMPILE);


15) Current waits in flight (requests)


SELECT TOP (50)
  r.session_id, r.status, r.wait_type, r.wait_time, r.wait_resource,
  DB_NAME(r.database_id) AS database_name,
  st.text
FROM sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id <> @@SPID
ORDER BY r.wait_time DESC
OPTION (RECOMPILE);


16) Waiting tasks detail (who waits on what)


SELECT TOP (50)
  wt.session_id, wt.wait_type, wt.wait_duration_ms, wt.resource_description,
  wt.blocking_session_id
FROM sys.dm_os_waiting_tasks wt
ORDER BY wt.wait_duration_ms DESC
OPTION (RECOMPILE);


17) Scheduler health (runnable queue / CPU)


SELECT
  scheduler_id, current_tasks_count, runnable_tasks_count, current_workers_count, active_workers_count
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE'
ORDER BY runnable_tasks_count DESC
OPTION (RECOMPILE);


18) CPU ring buffer history (last N minutes)

*(same idea as your script, parameterized)*


DECLARE @Duration INT = 240;
DECLARE @ts_now BIGINT = (SELECT cpu_ticks/(cpu_ticks/ms_ticks) FROM sys.dm_os_sys_info);

SELECT TOP (@Duration)
  SQLProcessUtilization,
  SystemIdle,
  100 - SystemIdle - SQLProcessUtilization AS OtherProcess,
  DATEADD(ms, -1*(@ts_now - [timestamp]), CURRENT_TIMESTAMP) AS EventTime
FROM (
  SELECT
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]','int') AS SystemIdle,
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]','int') AS SQLProcessUtilization,
    [timestamp]
  FROM (
    SELECT CONVERT(xml, record) AS record, [timestamp]
    FROM sys.dm_os_ring_buffers
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
      AND record LIKE N'%<SystemHealth>%'
  ) x
) y
ORDER BY EventTime DESC
OPTION (RECOMPILE);


19) Spinlock stats (advanced)


SELECT TOP (50)
  name, collisions, spins, spins_per_collision
FROM sys.dm_os_spinlock_stats
ORDER BY collisions DESC
OPTION (RECOMPILE);


20) Latch waits (tempdb / allocation hints)


SELECT TOP (50)
  wait_type, waiting_requests_count, wait_time_ms
FROM sys.dm_os_latch_stats
ORDER BY wait_time_ms DESC
OPTION (RECOMPILE);


21) IO pending requests


SELECT TOP (50)
  io_type, io_pending_ms_ticks, scheduler_address, io_handle, io_offset
FROM sys.dm_io_pending_io_requests
ORDER BY io_pending_ms_ticks DESC
OPTION (RECOMPILE);


22) Resource Governor stats (if used)


SELECT * 
FROM sys.dm_resource_governor_workload_groups
OPTION (RECOMPILE);



 Blocking, Locks, Deadlocks (23–32)

23) Full blocking tree (classic triage)


SELECT
  r.session_id,
  r.blocking_session_id,
  r.status,
  r.wait_type,
  r.wait_time,
  r.wait_resource,
  DB_NAME(r.database_id) AS database_name,
  st.text
FROM sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.session_id <> @@SPID
ORDER BY r.blocking_session_id DESC, r.wait_time DESC
OPTION (RECOMPILE);


24) Blocked sessions only


SELECT
  r.session_id,
  r.blocking_session_id,
  r.wait_type,
  r.wait_time,
  r.wait_resource,
  st.text AS blocked_sql
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id <> 0
OPTION (RECOMPILE);


25) Who is blocking others (top blockers)


SELECT TOP (50)
  r.blocking_session_id AS blocker_spid,
  COUNT(*) AS blocked_count
FROM sys.dm_exec_requests r
WHERE r.blocking_session_id <> 0
GROUP BY r.blocking_session_id
ORDER BY blocked_count DESC
OPTION (RECOMPILE);


26) Locks by session


SELECT TOP (50)
  request_session_id,
  resource_type,
  request_mode,
  request_status,
  resource_database_id
FROM sys.dm_tran_locks
ORDER BY request_session_id
OPTION (RECOMPILE);


27) Open transactions (who forgot to commit)


SELECT
  s.session_id, s.login_name, s.host_name, s.program_name,
  at.transaction_begin_time, at.transaction_type, at.transaction_state
FROM sys.dm_tran_session_transactions st
JOIN sys.dm_tran_active_transactions at ON st.transaction_id = at.transaction_id
JOIN sys.dm_exec_sessions s ON st.session_id = s.session_id
ORDER BY at.transaction_begin_time
OPTION (RECOMPILE);


28) Requests + isolation levels


SELECT
  r.session_id,
  r.transaction_isolation_level,
  CASE r.transaction_isolation_level
    WHEN 0 THEN 'Unspecified' WHEN 1 THEN 'ReadUncommitted'
    WHEN 2 THEN 'ReadCommitted' WHEN 3 THEN 'Repeatable'
    WHEN 4 THEN 'Serializable' WHEN 5 THEN 'Snapshot' END AS isolation_desc,
  r.open_transaction_count
FROM sys.dm_exec_requests r
WHERE r.session_id <> @@SPID
OPTION (RECOMPILE);


29) Deadlock info (system_health XE, 2012+)


SELECT TOP (50)
  XEventData.XEvent.value('(event/@timestamp)[1]','datetime2') AS [timestamp],
  XEventData.XEvent.query('.') AS deadlock_graph
FROM (
  SELECT CAST(target_data AS xml) AS TargetData
  FROM sys.dm_xe_session_targets t
  JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
  WHERE s.name = 'system_health'
    AND t.target_name = 'ring_buffer'
) AS Data
CROSS APPLY TargetData.nodes('//event[@name="xml_deadlock_report"]') AS XEventData(XEvent)
ORDER BY [timestamp] DESC;


30) Tempdb version store pressure (snapshot/RCSI)


SELECT
  SUM(version_store_reserved_page_count)*8/1024.0 AS version_store_mb,
  SUM(internal_object_reserved_page_count)*8/1024.0 AS internal_objects_mb,
  SUM(user_object_reserved_page_count)*8/1024.0 AS user_objects_mb
FROM sys.dm_db_file_space_usage
OPTION (RECOMPILE);


31) Tempdb sessions using space


SELECT TOP (50)
  session_id,
  (user_objects_alloc_page_count - user_objects_dealloc_page_count)*8/1024.0 AS user_objects_mb,
  (internal_objects_alloc_page_count - internal_objects_dealloc_page_count)*8/1024.0 AS internal_objects_mb
FROM sys.dm_db_session_space_usage
ORDER BY (internal_objects_alloc_page_count - internal_objects_dealloc_page_count) DESC
OPTION (RECOMPILE);


32) Active sessions + connection info


SELECT TOP (50)
  s.session_id, s.login_name, s.host_name, s.program_name, s.status,
  c.client_net_address, c.connect_time
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
WHERE s.is_user_process = 1
ORDER BY s.session_id
OPTION (RECOMPILE);




 Disk / IO Latency (33–40)

33) File-level IO latency (reads/writes)


SELECT TOP (50)
  DB_NAME(vfs.database_id) AS database_name,
  mf.physical_name,
  vfs.num_of_reads,
  vfs.io_stall_read_ms,
  CASE WHEN vfs.num_of_reads = 0 THEN 0 ELSE vfs.io_stall_read_ms*1.0/vfs.num_of_reads END AS avg_read_ms,
  vfs.num_of_writes,
  vfs.io_stall_write_ms,
  CASE WHEN vfs.num_of_writes = 0 THEN 0 ELSE vfs.io_stall_write_ms*1.0/vfs.num_of_writes END AS avg_write_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) vfs
JOIN sys.master_files mf
  ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id
ORDER BY (vfs.io_stall_read_ms + vfs.io_stall_write_ms) DESC
OPTION (RECOMPILE);


34) Volume free space (mount points)


SELECT
  vs.volume_mount_point,
  MAX(vs.total_bytes)/1024.0/1024.0/1024.0 AS total_gb,
  MAX(vs.available_bytes)/1024.0/1024.0/1024.0 AS free_gb
FROM sys.master_files f
CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id) vs
GROUP BY vs.volume_mount_point
ORDER BY free_gb DESC
OPTION (RECOMPILE);


35) Pending IO requests


SELECT *
FROM sys.dm_io_pending_io_requests
ORDER BY io_pending_ms_ticks DESC
OPTION (RECOMPILE);


36) Database log usage (per DB)


SELECT
  d.name,
  ls.total_log_size_mb,
  ls.used_log_space_mb,
  ls.used_log_space_in_percent
FROM sys.databases d
CROSS APPLY sys.dm_db_log_space_usage ls
ORDER BY ls.used_log_space_in_percent DESC
OPTION (RECOMPILE);


37) Databases with long log flush waits (symptom)


SELECT TOP (50)
  DB_NAME(database_id) AS database_name,
  total_log_flush_count,
  total_log_flush_wait_time_ms,
  CASE WHEN total_log_flush_count = 0 THEN 0 ELSE total_log_flush_wait_time_ms*1.0/total_log_flush_count END AS avg_log_flush_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL)
ORDER BY avg_log_flush_ms DESC
OPTION (RECOMPILE);


38) Checkpoint and lazy writer pressure (perf counters DMV)


SELECT counter_name, cntr_value
FROM sys.dm_os_performance_counters
WHERE object_name LIKE '%Buffer Manager%'
  AND counter_name IN ('Checkpoint pages/sec','Lazy writes/sec','Free list stalls/sec')
OPTION (RECOMPILE);


39) Page Life Expectancy


SELECT cntr_value AS PLE_seconds
FROM sys.dm_os_performance_counters
WHERE object_name LIKE '%Buffer Manager%'
  AND counter_name = 'Page life expectancy'
OPTION (RECOMPILE);


40) IO stats by database (summary)


SELECT TOP (50)
  DB_NAME(vfs.database_id) AS database_name,
  SUM(vfs.num_of_reads) AS reads,
  SUM(vfs.num_of_writes) AS writes,
  SUM(vfs.io_stall) AS total_io_stall_ms
FROM sys.dm_io_virtual_file_stats(NULL,NULL) vfs
GROUP BY vfs.database_id
ORDER BY total_io_stall_ms DESC
OPTION (RECOMPILE);




 Instance Baselines (41–50)

41) Instance uptime (start time)


SELECT sqlserver_start_time
FROM sys.dm_os_sys_info
OPTION (RECOMPILE);


42) SQL + OS version


SELECT @@VERSION AS version_info;


43) Server properties (edition/build/etc.)


SELECT
  SERVERPROPERTY('MachineName') AS machine_name,
  SERVERPROPERTY('ServerName') AS server_name,
  SERVERPROPERTY('InstanceName') AS instance_name,
  SERVERPROPERTY('Edition') AS edition,
  SERVERPROPERTY('ProductVersion') AS product_version,
  SERVERPROPERTY('ProductLevel') AS product_level,
  SERVERPROPERTY('IsHadrEnabled') AS is_hadr_enabled;


44) Memory clerks (where memory went)


SELECT TOP (50)
  type,
  SUM(pages_kb)/1024.0 AS mb
FROM sys.dm_os_memory_clerks
GROUP BY type
ORDER BY mb DESC
OPTION (RECOMPILE);


45) Buffer descriptors (total)


SELECT COUNT(*) AS buffer_pages,
       COUNT(*)*8/1024.0 AS buffer_mb
FROM sys.dm_os_buffer_descriptors
OPTION (RECOMPILE);


46) Max/Min server memory config


SELECT name, value_in_use
FROM sys.configurations
WHERE name IN ('min server memory (MB)','max server memory (MB)')
OPTION (RECOMPILE);


47) Server memory targets vs actual (perf counters DMV)


SELECT counter_name, cntr_value/1024.0/1024.0 AS value_gb
FROM sys.dm_os_performance_counters
WHERE counter_name IN ('Total Server Memory (KB)','Target Server Memory (KB)')
OPTION (RECOMPILE);


48) Connection count


SELECT COUNT(*) AS total_connections
FROM sys.dm_exec_connections
OPTION (RECOMPILE);


49) Database states


SELECT name, state_desc, recovery_model_desc
FROM sys.databases
ORDER BY name;


50) SQL Server memory dumps (if any)


SELECT
  filename,
  creation_time,
  size_in_bytes/1024.0/1024.0 AS size_mb
FROM sys.dm_server_memory_dumps
ORDER BY creation_time DESC
OPTION (RECOMPILE);



 Closing Note:
This repository is a curated collection of SQL Server DMV scripts I use in real-world troubleshooting and daily DBA health checks.  
Use these queries as a starting point - always validate findings with system context (workload patterns, recent deployments, maintenance jobs, and baseline metrics).

⚠️ Important: Some queries may be resource-intensive on very busy production systems. When in doubt:
- Run during low-traffic windows
- Filter by database/session where possible
- Prefer sampling over repeated full scans

If you find improvements or want to contribute additional scripts, feel free to open a pull request.

— Teddy
















