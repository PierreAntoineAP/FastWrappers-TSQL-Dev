# FastWrappers-TSQL

A project to wrap FastTransfer and FastBCP in a CLR Assembly and allow to call them from T-SQL using extended store procedure.
As a reminder :
- **FastTransfer** is a CLI that allow import from file or transfer data between databases using streaming and parallel mecanism for high performance
- **FastBCP** is a CLI that allow to export data from databases to files (csv, parquet, json,bson and excel) using streaming and parallel mecanism for high performance

## Installation

Download the latest release from the [Releases page](https://github.com/aetperf/FastWrappers-TSQL/releases). Each release provides 4 installation options:

1. **FastWrappers-TSQL.dacpac** - Data-tier Application Package (recommended for Visual Studio / SQL Server Data Tools)
2. **FastWrappers-TSQL.bacpac** - Binary Application Package (for import/export between servers)
3. **FastWrappers-TSQL.bak** - SQL Server Backup file (compatible with SQL Server 2016+, restore using SSMS)
4. **FastWrappers-TSQL.sql** - Pure SQL Script (execute using sqlcmd or SSMS)

### Post-Installation Configuration

After restoring the database (especially from .bak), you **must** configure CLR integration. Two options are available depending on your environment:

#### Option 1: Using sp_add_trusted_assembly (Recommended for Production) 🔒

This approach is **more secure** as it keeps TRUSTWORTHY OFF and doesn't require changing the database owner:

```sql
-- Enable CLR integration
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
GO

EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;
GO

-- Extract the assembly hash and add it to trusted assemblies
DECLARE @hash VARBINARY(64);

SELECT @hash = HASHBYTES('SHA2_512', content)
FROM sys.assembly_files
WHERE assembly_id = (
    SELECT assembly_id 
    FROM sys.assemblies 
    WHERE name = 'FastWrappers_TSQL'
);

EXEC sys.sp_add_trusted_assembly 
    @hash = @hash,
    @description = N'FastWrappers_TSQL Assembly v0.7.0';
GO
```

**Advantages:**
- ✅ TRUSTWORTHY remains OFF (more secure)
- ✅ No need to change database owner
- ✅ Only this specific assembly is trusted

**Note:** The assembly hash changes with each version. When upgrading, you must remove the old hash and add the new one:
```sql
-- Remove old version
EXEC sys.sp_drop_trusted_assembly @hash = <old_hash>;
-- Then run the sp_add_trusted_assembly script above
```

#### Option 2: Using TRUSTWORTHY ON (Quick Setup for Dev/Test)

This approach is simpler but **less secure**. Use only for development/testing environments:

```sql
-- Enable TRUSTWORTHY for signed UNSAFE assemblies
ALTER DATABASE [FastWrappers-TSQL] SET TRUSTWORTHY ON;
GO

-- Enable CLR integration
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
GO

EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;
GO

-- Set database owner to 'sa' (required for signed UNSAFE assemblies)
EXEC sp_changedbowner 'sa';
GO
```

**Important:** With this method, the `sp_changedbowner 'sa'` command is **critical**. Without it, you will encounter error 0x80FC80F1 when trying to execute the stored procedures.

## Usage Examples

### Copy one table using 12 threads between two MSSQL instances 
```TSQL
-- use SELECT [dbo].[EncryptString]('<YourPassWordToEncrypt>') to get the encrypted password
EXEC dbo.xp_RunFastTransfer_secure
     @fastTransferDir='C:\FastTransfert\win-x64\latest\',
     @sourceConnectionType = N'mssql',
     @sourceServer = N'localhost',
     @sourceUser = N'FastUser',
     @sourcePasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
     @sourceDatabase = N'tpch_test',
     @sourceSchema = N'dbo',
     @sourceTable = N'orders',
     @targetConnectionType = N'msbulk',
     @targetServer = N'localhost\SS2025',
     @targetUser = N'FastUser',
     @targetPasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
     @targetDatabase = N'tpch_test',
     @targetSchema = N'dbo',
     @targetTable = N'orders_3',
     @loadMode = N'Truncate',
     @batchSize = 130000,
     @method = N'RangeId',
     @distributeKeyColumn = N'o_orderkey',
     @degree = 12,
     @mapmethod = 'Name',
     @runId = N'CLRWrap_Run_MS2MS_20250328'
```

### Copy one table using 12 threads between an Oracle database and SQL instance 
```TSQL
-- use SELECT [dbo].[EncryptString]('<YourPassWordToEncrypt>') to get the encrypted password

EXEC dbo.xp_RunFastTransfer_secure
	@fastTransferDir = 'C:\FastTransfer\win-x64\latest\',
    @sourceConnectionType = 'mssql',
	@sourceServer = 'localhost',
	@sourceUser = 'FastUser',
	@sourcePasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
	@sourceDatabase = 'tpch_test',
	@sourceSchema = 'dbo',
	@sourceTable = 'orders',
	@targetConnectionType = 'msbulk',
	@targetServer = 'localhost\SS2025',
	@targetUser = 'FastUser',
	@targetPasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
	@targetDatabase = 'tpch_test',
	@targetSchema = 'dbo',
	@targetTable = 'orders_3',
	@loadmode = 'Truncate',
	@batchSize = 130000,
	@method = 'RangeId',
	@distributeKeyColumn = 'o_orderkey',
	@degree = 12,
	@mapmethod = 'Name',
	@runId = 'test_MSSQL_to_MSSQL_P12_RangeId'
     @mapmethod = 'Name',
     @runId = N'CLRWrap_Run_ORA2MS_20250328'
```

### Export one table to a csv file mono thread 
```TSQL
-- use SELECT [dbo].[EncryptString]('<YourPassWordToEncrypt>') to get the encrypted password

EXEC dbo.xp_RunFastBCP_secure
   @fastBCPDir = 'D:\FastBCP\latest',
   @connectionType = 'mssql',
   @sourceserver = 'localhost',
   @sourceuser = 'FastUser',
   @sourceschema = 'dbo',
   @sourcetable = 'orders',
   @sourcepasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
   @sourcedatabase = 'tpch',
   @query = 'SELECT top 1000 * FROM orders',
   @outputFile = 'orders_output.csv',
   @outputDirectory = 'D:\temp\fastbcpoutput\{sourcedatabase}\{sourceschema}\{sourcetable}',
   @delimiter = '|',
   @usequotes = 1,
   @dateformat = 'yyyy-MM-dd HH24:mm:ss',
   @encoding = 'utf-8',
   @method = 'None',
   @runid = 'test_FastBCP_export_orders'
```

### Export one table to 8 parquet files using 8 threads 
```TSQL
-- use SELECT [dbo].[EncryptString]('<YourPassWordToEncrypt>') to get the encrypted password
EXEC dbo.xp_RunFastBCP_secure
   @fastBCPDir = 'D:\FastBCP\latest',
   @connectionType = 'mssql',
   @sourceserver = 'localhost',
   @sourceuser = 'FastUser',
   @sourceschema = 'dbo',
   @sourcetable = 'orders_15M',
   @sourcepasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
   @sourcedatabase = 'tpch10_collation_bin2',
   @outputFile = 'orders_output.parquet',
   @outputDirectory = 'D:\temp\fastbcpoutput\{sourcedatabase}\{sourceschema}\{sourcetable}',
   @method = 'Ntile',
   @distributeKeyColumn = 'o_orderkey',    
   @degree = 8,
   @mergeDistributedFile = 0
   ```


### Export one table several csv files (one file by month) using 8 threads 
```TSQL
-- use SELECT [dbo].[EncryptString]('<YourPassWordToEncrypt>') to get the encrypted password
EXEC dbo.xp_RunFastBCP_secure
   @fastBCPDir = 'D:\FastBCP\latest',
   @connectionType = 'mssql',
   @sourceserver = 'localhost',
   @sourceuser = 'FastUser',
   @sourceschema = 'dbo',
   @sourcetable = 'orders_15M',
   @sourcepasswordSecure = 'wi1/VHz9s+fp45186iLYYQ==',
   @sourcedatabase = 'tpch10_collation_bin2',
   @query = 'SELECT * FROM (SELECT *, year(o_orderdate)*100+month(o_orderdate) o_ordermonth from orders_15M) src',
   @outputFile = 'orders_output.csv',
   @outputDirectory = 'D:\temp\fastbcpoutput\{sourcedatabase}\{sourceschema}\{sourcetable}',
   @method = 'DataDriven',
   @distributeKeyColumn = 'o_ordermonth',    
   @degree = 8,
   @mergeDistributedFile = 0
   ```


## Nota :
You must have a valid trial or a valid FastTransfer.exe (or FastTransfer binary for linux) into the directory you specified with @fastTransferDir. The sql server service user must have read/execute provilege on the directory and FastTransfer(.exe) file
