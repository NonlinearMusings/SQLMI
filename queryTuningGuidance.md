# Query Tuning with Managed Instance

Ideally, before deploying to MI a [performance
baseline](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance-migrate#create-performance-baseline)
is created, in part to establish the appropriate [Service
Tier](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance#managed-instance-service-tiers).

The *”performance baseline is a set of parameters such as average/max CPU usage,
average/max disk IO latency, throughput, IOPS, average/max page life expectancy,
average max size of tempdb. You would like to have similar or even better
parameters after migration, so it is important to measure and record the
baseline values for these parameters. In addition to system parameters, you
would need to select a set of the representative queries or the most important
queries in your workload and measure min/average/max duration, CPU usage for the
selected queries. These values would enable you to compare performance of
workload running on Managed Instance to the original values on your source SQL
Server.”*

The following [Key causes of performance differences between SQL managed
instance and SQL
Server](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Fazure.microsoft.com%2Fen-us%2Fblog%2Fkey-causes-of-performance-differences-between-sql-managed-instance-and-sql-server%2F&data=02%7C01%7CRick.Davis%40microsoft.com%7Cc7376f5b1fa2493b1db508d7e073a9c8%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637224659682405934&sdata=sOmy3GH6u25X6xQPXV82FXIuA8pAh6keUqGpyOVKE8Y%3D&reserved=0)
should also be taken into consideration as a simple 1:1 correlation with a VM’s
configuration is not possible.

Similarly, MI is always running a newer version of SQL Server than is available
for VM installation. As such, [Compatibility levels and Database Engine
upgrades](https://docs.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-compatibility-level?view=sql-server-ver15#compatibility-levels-and-database-engine-upgrades)
should be considered for existing SQL Server databases being migrated to MI as
changes in query plans can be either beneficial or detrimental, and the only way
to know is to set the desired compatibility level and test.

Additionally, for migrated databases, MI’s MAXDOP setting should be crosschecked
with the source’s setting. The assumption is that the combination of database
compatibility level and MAXDOP will produce equivalent query performance times.
However, these initial settings are just that: A starting point. Testing should
be performed with the most recent database compatibility level and with MAXDOP =
0 to see if the latest version of MI’s engine enhancements and query optimizer
benefit the queries as-is or not. If not, do not rule out the possibility of
refactoring/tuning the query to see better performance.

If query performance is still wanting, then it is time to start looking at where
the waits are. [Database Advisor performance recommendations for single and
pooled
databases](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-advisor)
offers a good starting point for identifying high-level issues, such as missing
indexes which can have significant impact on performance. [Query Performance
Insight for Azure SQL
Database](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-query-performance)
should be used to identify the top resource consuming queries as frequently the
most resource intensive queries are the ones to prioritize for tuning. Next, we
need to see what the actual waits are and there are multiple resources for
determining them. The DMV
[sys.dm_db_wait_stats](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-wait-stats-azure-sql-database?view=azuresqldb-current)
“returns information about all the waits encountered by threads that executed
during operation”, meaning it’s a real time looks the database. Paul Randal’s
excellent [SQL Server Wait Statistics (or please tell me where it
hurts…)](https://www.sqlskills.com/blogs/paul/wait-statistics-or-please-tell-me-where-it-hurts/)
will return a listing of wait types being experienced by SQL Server, in
decreasing order, thereby allowing you to quickly determine server-level issues
affecting all queries. (Individual, query-level tuning should be performed using
SSMS’ [Actual Execution
Plan](https://docs.microsoft.com/en-us/sql/relational-databases/performance/display-an-actual-execution-plan?view=sql-server-ver15)
capability.)

[Troubleshoot Azure SQL Database performance issues with Intelligent
Insights](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-intelligent-insights-troubleshoot-performance)
may also be used to assist with database performance tuning activities.

However, if these initial steps do not yield sufficient results, then it is time
to dive into MI’s storage layer…

The first takeaway needs to be that MI’s storage layer does not map to a VM’s;
especially if its storage is SAN based and expects caching support.
Additionally, the physical storage implementation changes between the General
Purpose and Business Critical tiers. (The following information can be found at
[Overview Azure SQL Database managed instance resource
limits](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance-resource-limits#service-tier-characteristics).)

Business Critical storage is implemented via local SSDs and scales linearly with
vCore count. BC enables 2500 IOPS/vCore up to a maximum of 200,000 IOPS with 80
vCores. Similarly, Log write throughput is 4 MB/sec/vCore up to a maximum of 48
MB/sec: both with a latency of 1-2ms.

General Purpose storage is both more restricted and nuanced. For starters, GP
storage is implemented via remote Azure Blob Storage (Premium) and as such has
latency times from 5-10 ms. GP IOPS and Throughput is a reflection of the number
and size of data files as shown in this table: [File IO characteristics in
General Purpose
tier](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance-resource-limits#file-io-characteristics-in-general-purpose-tier).
However, IOPS are capped at a theoretical maximum of 40,000. Log write
throughput is also reflective of the log file’s size, up to a maximum of 22
MB/sec.

Why all the vague talk for GP IOPS? Well, because of this: [Storage performance
best practices and considerations for Azure SQL DB Managed Instance (General
Purpose)](https://techcommunity.microsoft.com/t5/datacat/storage-performance-best-practices-and-considerations-for-azure/ba-p/305525).
Without restating the article, optimizing IOPS in GP is the science of working
out the optimal data file size and count for all databases within in the
Instance, including system database and TempDB – which in total cannot exceed 8
TB of data while also not exceeding the underlying storage allocation maximum of
35 TB.

As a simplistic example: Assume GP hosting a single database configured for
maximum read and write IOPS. Allocating 6x 1.1 TB data files would provision 6x
P40 disks with a total of 45,000 IOPS and 1.5 GB/sec throughput for data. Adding
a 1.1 TB log file provisions a P40 disk with 7,500 IOPS and 250 MB/sec (capped
at 22 MB/sec). Thus, 7.7 TB of data space has been allocated, consuming 14TB of
storage, leaving 300 GB remaining for TempDB/TempDB-Log and System databases –
up to the 8 TB maximum. However, since the storage for this instance has been
maxed out for the sake of this one database no room remains to host additional
databases in this instance. If additional databases need to be hosted on this
instance, then the data file sizes must be reduced, and their associated
performance benefits will also reduce in kind.

Additional readings:

-   [Storage layer in General Purpose Azure SQL Managed
    Instance](https://medium.com/azure-sqldb-managed-instance/file-layout-in-general-purpose-azure-sql-managed-instance-cf21fff9c76c)

-   [Increasing data files might improve performance on General Purpose Managed
    Instance
    tier](https://medium.com/azure-sqldb-managed-instance/increasing-data-files-might-improve-performance-on-general-purpose-managed-instance-tier-6e90bad2ae4b)

-   [Impact of log file size on General Purpose Managed Instance
    performance](https://medium.com/azure-sqldb-managed-instance/impact-of-log-file-size-on-general-purpose-managed-instance-performance-21ad170c823e)
