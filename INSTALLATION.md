# FastWrappers-TSQL Installation Guide

## 🔒 Secure Installation (Recommended)

FastWrappers-TSQL uses **certificate-based security** to eliminate the need for `TRUSTWORTHY ON`, following SQL Server security best practices.

### Prerequisites

- SQL Server 2016 or later
- Sysadmin privileges for initial certificate installation
- CLR enabled on the SQL Server instance

---

## Step 1: One-Time Server Setup (Certificate Installation)

**This step is performed ONCE per SQL Server instance.**

### 1.1 Download Required Files

From the [latest release](https://github.com/PierreAntoineAP/FastWrappers-TSQL-Dev/releases/latest), download:
- `FastWrappers-Certificate.cer`
- `Install-Certificate.sql`

### 1.2 Place the Certificate

Copy `FastWrappers-Certificate.cer` to a secure location on your server, for example:
- `C:\Certificates\FastWrappers-Certificate.cer`

### 1.3 Update and Run Installation Script

1. Open `Install-Certificate.sql` in SQL Server Management Studio
2. Update the file path on line 24:
   ```sql
   CREATE CERTIFICATE FastWrappersCert
   FROM FILE = 'C:\Certificates\FastWrappers-Certificate.cer';  -- Update this path
   ```
3. Execute the script as **sysadmin**

**What this does:**
- Creates a certificate in `master` database
- Creates a login trusted for UNSAFE assemblies
- Grants necessary permissions

### 1.4 Enable CLR (If Not Already Enabled)

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
GO

EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;
GO
```

✅ **Server setup complete!** You can now deploy FastWrappers-TSQL to any database on this server.

---

## Step 2: Deploy FastWrappers-TSQL Database

After the certificate is installed, choose one of the following deployment methods:

### Option A: Restore from Backup (.bak)

```sql
RESTORE DATABASE [FastWrappers-TSQL]
FROM DISK = N'C:\Path\To\FastWrappers-TSQL.bak'
WITH MOVE 'FastWrappers-TSQL' TO 'C:\SQLData\FastWrappers-TSQL.mdf',
     MOVE 'FastWrappers-TSQL_log' TO 'C:\SQLData\FastWrappers-TSQL_log.ldf',
     REPLACE;
GO
```

### Option B: Deploy DACPAC (Visual Studio/SqlPackage)

```powershell
SqlPackage.exe /Action:Publish `
    /SourceFile:"FastWrappers-TSQL.dacpac" `
    /TargetServerName:"YourServer" `
    /TargetDatabaseName:"FastWrappers-TSQL"
```

### Option C: Import BACPAC

1. Open SQL Server Management Studio
2. Right-click on **Databases** → **Import Data-tier Application**
3. Select `FastWrappers-TSQL.bacpac`
4. Follow the wizard

### Option D: Execute SQL Script

```sql
-- Run FastWrappers-TSQL.sql in SQLCMD mode or SSMS
:r C:\Path\To\FastWrappers-TSQL.sql
```

---

## Step 3: Verify Installation

```sql
USE [FastWrappers-TSQL];
GO

-- Test encryption function
SELECT dbo.EncryptString('test123') AS EncryptedValue;
GO

-- Should return an encrypted base64 string
```

If this works without errors, installation is successful! ✅

---

## Security Benefits

✅ **No TRUSTWORTHY ON required** - More secure database configuration  
✅ **Certificate-based security** - Industry best practice  
✅ **Server-level trust** - Deploy to multiple databases easily  
✅ **Audit-friendly** - Clear security chain of trust  

---

## Troubleshooting

### Error: "Assembly may not be trusted" (0x80FC80F1)

**Cause:** Certificate not installed on the server.

**Solution:** Complete Step 1 (Certificate Installation)

### Error: "Could not load file or assembly"

**Cause:** CLR not enabled.

**Solution:**
```sql
EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;
```

### Error: "CREATE CERTIFICATE failed"

**Cause:** File path incorrect or insufficient permissions.

**Solution:** 
- Verify the certificate file path
- Ensure SQL Server service account can read the file
- Run as sysadmin

---

## Updating to a New Version

When a new version is released:

1. **If the certificate hasn't changed:** Simply deploy the new database files (no server setup needed)
2. **If a new certificate is included:** Re-run Step 1 with the new certificate

---

## Support

For issues or questions, please open an issue on [GitHub](https://github.com/PierreAntoineAP/FastWrappers-TSQL-Dev/issues).
