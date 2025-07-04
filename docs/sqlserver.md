<!--
GENERATED FILE - DO NOT EDIT
This file was generated by [MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets).
Source File: /docs/mdsource/sqlserver.source.md
To change this file edit the source file and then run MarkdownSnippets.
-->

# SQL Server usage

[![NuGet Status](https://img.shields.io/nuget/v/Delta.svg?label=Delta)](https://www.nuget.org/packages/Delta/)
[![NuGet Status](https://img.shields.io/nuget/v/Delta.SqlServer.svg?label=Delta.SqlServer)](https://www.nuget.org/packages/Delta.SqlServer/)

Docs for when using when using [SQL Server SqlClient](https://github.com/dotnet/SqlClient).


## Implementation

For SQL Server the transaction log is used (via [dm_db_log_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-stats-transact-sql)) if the current user has the `VIEW SERVER STATE` permission.<!-- include: sqlserver-implemenation. path: /docs/mdsource/sqlserver-implemenation.include.md -->

If `VIEW SERVER STATE` is not allowed then a combination of [Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/track-data-changes-sql-server) and/or [Row Versioning](https://learn.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql) is used.

Give the above certain kinds of operations will be detected:

|             | Transaction Log | Change Tracking | Row Versioning | Change Tracking<br>and Row Versioning |
|-------------|:---------------:|:---------------:|:--------------:|:----------------------------------:|
| Insert      |        ✅      |        ✅       |        ✅     |                  ✅                |
| Update      |        ✅      |        ✅       |        ✅     |                  ✅                |
| Hard Delete |        ✅      |        ✅       |        ❌     |                  ✅                |
| Soft Delete |        ✅      |        ✅       |        ✅     |                  ✅                |
| Truncate    |        ✅      |        ❌       |        ❌     |                  ❌                |
<!-- endInclude -->


## Timestamp calculation

<!-- include: sqlserver-timestamp. path: /docs/mdsource/sqlserver-timestamp.include.md -->
### `VIEW SERVER STATE` permission

Transaction log is used via [dm_db_log_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-stats-transact-sql).

<!-- snippet: SqlServerTimeStampWithServerState -->
<a id='snippet-SqlServerTimeStampWithServerState'></a>
```cs
select log_end_lsn
from sys.dm_db_log_stats(db_id())
```
<sup><a href='/src/Delta/DeltaExtensions_Sql.cs#L79-L82' title='Snippet source file'>snippet source</a> | <a href='#snippet-SqlServerTimeStampWithServerState' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### No `VIEW SERVER STATE` permission

A combination of [change_tracking_current_version](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/change-tracking-current-version-transact-sql) (if tracking is enabled) and [@@DBTS (row version timestamp)](https://learn.microsoft.com/en-us/sql/t-sql/functions/dbts-transact-sql)

<!-- snippet: SqlServerTimeStampNoServerState -->
<a id='snippet-SqlServerTimeStampNoServerState'></a>
```cs
declare @changeTracking bigint = change_tracking_current_version();
declare @timeStamp bigint = convert(bigint, @@dbts);

if (@changeTracking is null)
  select cast(@timeStamp as varchar)
else
  select cast(@timeStamp as varchar) + '-' + cast(@changeTracking as varchar)
```
<sup><a href='/src/Delta/DeltaExtensions_Sql.cs#L99-L107' title='Snippet source file'>snippet source</a> | <a href='#snippet-SqlServerTimeStampNoServerState' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->
<!-- endInclude -->


## Usage


### Example SQL schema

<!-- snippet: Usage.Schema.verified.sql -->
<a id='snippet-Usage.Schema.verified.sql'></a>
```sql
-- Tables

CREATE TABLE [dbo].[Companies](
	[Id] [uniqueidentifier] NOT NULL,
	[RowVersion] [timestamp] NOT NULL,
	[Content] [nvarchar](max) NULL,
 CONSTRAINT [PK_Companies] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

ALTER TABLE [dbo].[Companies] ENABLE CHANGE_TRACKING WITH(TRACK_COLUMNS_UPDATED = OFF)

CREATE TABLE [dbo].[Employees](
	[Id] [uniqueidentifier] NOT NULL,
	[RowVersion] [timestamp] NOT NULL,
	[CompanyId] [uniqueidentifier] NOT NULL,
	[Content] [nvarchar](max) NULL,
	[Age] [int] NOT NULL,
 CONSTRAINT [PK_Employees] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

CREATE NONCLUSTERED INDEX [IX_Employees_CompanyId] ON [dbo].[Employees]
(
	[CompanyId] ASC
) ON [PRIMARY]
```
<sup><a href='/src/DeltaTests/Usage.Schema.verified.sql#L1-L30' title='Snippet source file'>snippet source</a> | <a href='#snippet-Usage.Schema.verified.sql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Add to WebApplicationBuilder

<!-- snippet: UseDeltaSqlServer -->
<a id='snippet-UseDeltaSqlServer'></a>
```cs
var builder = WebApplication.CreateBuilder();
builder.Services.AddScoped(_ => new SqlConnection(connectionString));
var app = builder.Build();
app.UseDelta();
```
<sup><a href='/src/WebApplicationSqlServer/Program.cs#L8-L15' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaSqlServer' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

### Add to a Route Group<!-- include: map-group. path: /docs/mdsource/map-group.include.md -->

To add to a specific [Route Group](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-handlers#route-groups):

<!-- snippet: UseDeltaMapGroup -->
<a id='snippet-UseDeltaMapGroup'></a>
```cs
app.MapGroup("/group")
    .UseDelta()
    .MapGet("/", () => "Hello Group!");
```
<sup><a href='/src/WebApplicationSqlServer/Program.cs#L61-L67' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaMapGroup' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->
<!-- endInclude -->


### ShouldExecute<!-- include: should-execute. path: /docs/mdsource/should-execute.include.md -->

Optionally control what requests Delta is executed on.

<!-- snippet: ShouldExecute -->
<a id='snippet-ShouldExecute'></a>
```cs
var app = builder.Build();
app.UseDelta(
    shouldExecute: httpContext =>
    {
        var path = httpContext.Request.Path.ToString();
        return path.Contains("match");
    });
```
<sup><a href='/src/DeltaTests/Usage.cs#L19-L29' title='Snippet source file'>snippet source</a> | <a href='#snippet-ShouldExecute' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->
<!-- endInclude -->


### Custom Connection discovery

By default, Delta uses `HttpContext.RequestServices` to discover the SqlConnection and SqlTransaction:

<!-- snippet: InitConnectionTypesSqlServer -->
<a id='snippet-InitConnectionTypesSqlServer'></a>
```cs
var sqlConnectionType = Type.GetType("Microsoft.Data.SqlClient.SqlConnection, Microsoft.Data.SqlClient");
if (sqlConnectionType != null)
{
    connectionType = sqlConnectionType;
    transactionType = sqlConnectionType.Assembly.GetType("Microsoft.Data.SqlClient.SqlTransaction")!;
    return;
}
```
<sup><a href='/src/Delta/DeltaExtensions_ConnectionDiscovery.cs#L12-L22' title='Snippet source file'>snippet source</a> | <a href='#snippet-InitConnectionTypesSqlServer' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

<!-- snippet: DiscoverConnection -->
<a id='snippet-DiscoverConnection'></a>
```cs
static Connection DiscoverConnection(HttpContext httpContext)
{
    var provider = httpContext.RequestServices;
    var connection = (DbConnection) provider.GetRequiredService(connectionType);
    var transaction = (DbTransaction?) provider.GetService(transactionType);
    return new(connection, transaction);
}
```
<sup><a href='/src/Delta/DeltaExtensions_ConnectionDiscovery.cs#L39-L49' title='Snippet source file'>snippet source</a> | <a href='#snippet-DiscoverConnection' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

To use custom connection discovery:

<!-- snippet: CustomDiscoveryConnectionSqlServer -->
<a id='snippet-CustomDiscoveryConnectionSqlServer'></a>
```cs
var application = webApplicationBuilder.Build();
application.UseDelta(
    getConnection: httpContext =>
        httpContext.RequestServices.GetRequiredService<SqlConnection>());
```
<sup><a href='/src/DeltaTests/Usage.cs#L315-L322' title='Snippet source file'>snippet source</a> | <a href='#snippet-CustomDiscoveryConnectionSqlServer' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

To use custom connection and transaction discovery:

<!-- snippet: CustomDiscoveryConnectionAndTransactionSqlServer -->
<a id='snippet-CustomDiscoveryConnectionAndTransactionSqlServer'></a>
```cs
var application = webApplicationBuilder.Build();
application.UseDelta(
    getConnection: httpContext =>
    {
        var provider = httpContext.RequestServices;
        var connection = provider.GetRequiredService<SqlConnection>();
        var transaction = provider.GetService<SqlTransaction>();
        return new(connection, transaction);
    });
```
<sup><a href='/src/DeltaTests/Usage.cs#L338-L350' title='Snippet source file'>snippet source</a> | <a href='#snippet-CustomDiscoveryConnectionAndTransactionSqlServer' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### GetLastTimeStamp:<!-- include: last-timestamp. path: /docs/mdsource/last-timestamp.include.md -->

`GetLastTimeStamp` is a helper method to get the DB timestamp that Delta uses to calculate the etag.

<!-- snippet: GetLastTimeStampConnection -->
<a id='snippet-GetLastTimeStampConnection'></a>
```cs
var timeStamp = await connection.GetLastTimeStamp();
```
<sup><a href='/src/DeltaTests/Usage.cs#L188-L192' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetLastTimeStampConnection' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->
<!-- endInclude -->


## Delta.SqlServer<!-- include: sqlserver-helpers. path: /docs/mdsource/sqlserver-helpers.include.md -->

A set of helper methods for working with [SQL Server Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/track-data-changes-sql-server) and [SQL Server Row Versioning](https://learn.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql)

Nuget: [Delta.SqlServer](https://www.nuget.org/packages/Delta.SqlServer)


### GetDatabasesWithTracking

Get a list of all databases with change tracking enabled.

<!-- snippet: GetDatabasesWithTracking -->
<a id='snippet-GetDatabasesWithTracking'></a>
```cs
var trackedDatabases = await sqlConnection.GetTrackedDatabases();
foreach (var db in trackedDatabases)
{
    Trace.WriteLine(db);
}
```
<sup><a href='/src/DeltaTests/Usage.cs#L204-L212' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetDatabasesWithTracking' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:

<!-- snippet: GetTrackedDatabasesSql -->
<a id='snippet-GetTrackedDatabasesSql'></a>
```cs
select d.name
from sys.databases as d inner join
  sys.change_tracking_databases as t on
  t.database_id = d.database_id
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L151-L156' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetTrackedDatabasesSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### GetTrackedTables

Get a list of all tracked tables in database.

<!-- snippet: GetTrackedTables -->
<a id='snippet-GetTrackedTables'></a>
```cs
var trackedTables = await sqlConnection.GetTrackedTables();
foreach (var db in trackedTables)
{
    Trace.WriteLine(db);
}
```
<sup><a href='/src/DeltaTests/Usage.cs#L230-L238' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetTrackedTables' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:

<!-- snippet: GetTrackedTablesSql -->
<a id='snippet-GetTrackedTablesSql'></a>
```cs
select t.Name
from sys.tables as t inner join
  sys.change_tracking_tables as c on t.[object_id] = c.[object_id]
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L81-L85' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetTrackedTablesSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### IsTrackingEnabled

Determine if change tracking is enabled for a database.

<!-- snippet: IsTrackingEnabled -->
<a id='snippet-IsTrackingEnabled'></a>
```cs
var isTrackingEnabled = await sqlConnection.IsTrackingEnabled();
```
<sup><a href='/src/DeltaTests/Usage.cs#L304-L308' title='Snippet source file'>snippet source</a> | <a href='#snippet-IsTrackingEnabled' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:

<!-- snippet: IsTrackingEnabledSql -->
<a id='snippet-IsTrackingEnabledSql'></a>
```cs
select count(d.name)
from sys.databases as d inner join
  sys.change_tracking_databases as t on
  t.database_id = d.database_id
where d.name = '{database}'
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L103-L109' title='Snippet source file'>snippet source</a> | <a href='#snippet-IsTrackingEnabledSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### EnableTracking

Enable change tracking for a database.

<!-- snippet: EnableTracking -->
<a id='snippet-EnableTracking'></a>
```cs
await sqlConnection.EnableTracking();
```
<sup><a href='/src/DeltaTests/Usage.cs#L298-L302' title='Snippet source file'>snippet source</a> | <a href='#snippet-EnableTracking' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:

<!-- snippet: EnableTrackingSql -->
<a id='snippet-EnableTrackingSql'></a>
```cs
alter database {database}
set change_tracking = on
(
  change_retention = {retentionDays} days,
  auto_cleanup = on
)
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L64-L71' title='Snippet source file'>snippet source</a> | <a href='#snippet-EnableTrackingSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### DisableTracking

Disable change tracking for a database and all tables within that database.

<!-- snippet: DisableTracking -->
<a id='snippet-DisableTracking'></a>
```cs
await sqlConnection.DisableTracking();
```
<sup><a href='/src/DeltaTests/Usage.cs#L283-L287' title='Snippet source file'>snippet source</a> | <a href='#snippet-DisableTracking' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:


#### For disabling tracking on a database:

<!-- snippet: DisableTrackingSqlDB -->
<a id='snippet-DisableTrackingSqlDB'></a>
```cs
alter database [{database}] set change_tracking = off;
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L135-L137' title='Snippet source file'>snippet source</a> | <a href='#snippet-DisableTrackingSqlDB' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### For disabling tracking on tables:

<!-- snippet: DisableTrackingSqlTable -->
<a id='snippet-DisableTrackingSqlTable'></a>
```cs
alter table [{table}] disable change_tracking;
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L126-L128' title='Snippet source file'>snippet source</a> | <a href='#snippet-DisableTrackingSqlTable' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### SetTrackedTables

Enables change tracking for all tables listed, and disables change tracking for all tables not listed.

<!-- snippet: SetTrackedTables -->
<a id='snippet-SetTrackedTables'></a>
```cs
await sqlConnection.SetTrackedTables(["Companies"]);
```
<sup><a href='/src/DeltaTests/Usage.cs#L224-L228' title='Snippet source file'>snippet source</a> | <a href='#snippet-SetTrackedTables' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Uses the following SQL:


#### For enabling tracking on a database:

<!-- snippet: EnableTrackingSql -->
<a id='snippet-EnableTrackingSql'></a>
```cs
alter database {database}
set change_tracking = on
(
  change_retention = {retentionDays} days,
  auto_cleanup = on
)
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L64-L71' title='Snippet source file'>snippet source</a> | <a href='#snippet-EnableTrackingSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### For enabling tracking on tables:

<!-- snippet: EnableTrackingTableSql -->
<a id='snippet-EnableTrackingTableSql'></a>
```cs
alter table [{table}] enable change_tracking
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L23-L25' title='Snippet source file'>snippet source</a> | <a href='#snippet-EnableTrackingTableSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### For disabling tracking on tables:

<!-- snippet: DisableTrackingTableSql -->
<a id='snippet-DisableTrackingTableSql'></a>
```cs
alter table [{table}] disable change_tracking;
```
<sup><a href='/src/Delta.SqlServer/DeltaExtensions_Sql.cs#L34-L36' title='Snippet source file'>snippet source</a> | <a href='#snippet-DisableTrackingTableSql' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->
<!-- endInclude -->
