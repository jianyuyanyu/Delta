<!--
GENERATED FILE - DO NOT EDIT
This file was generated by [MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets).
Source File: /readme.source.md
To change this file edit the source file and then run MarkdownSnippets.
-->

# <img src="/src/icon.png" height="30px"> Delta

[![Build status](https://ci.appveyor.com/api/projects/status/20t96gnsmysklh09/branch/main?svg=true)](https://ci.appveyor.com/project/SimonCropp/Delta)
[![NuGet Status](https://img.shields.io/nuget/v/Delta.svg?label=Delta)](https://www.nuget.org/packages/Delta/)
[![NuGet Status](https://img.shields.io/nuget/v/Delta.EF.svg?label=Delta.EF)](https://www.nuget.org/packages/Delta.EF/)
[![NuGet Status](https://img.shields.io/nuget/v/Delta.SqlServer.svg?label=Delta.SqlServer)](https://www.nuget.org/packages/Delta.SqlServer/)

Delta is an approach to implementing a [304 Not Modified](https://www.keycdn.com/support/304-not-modified) leveraging DB change tracking.

The approach uses a last updated timestamp from the database to generate an [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag). All dynamic requests then have that ETag checked/applied.

This approach works well when the frequency of updates is relatively low. In this scenario, the majority of requests will leverage the result in a 304 Not Modified being returned and the browser loading the content its cache.

Effectively consumers will always receive the most current data, while the load on the server remains low.

**See [Milestones](../../milestones?state=closed) for release notes.**


## Powered by

[![JetBrains logo.](https://resources.jetbrains.com/storage/products/company/brand/logos/jetbrains.svg)](https://jb.gg/OpenSourceSupport)


## Assumptions

Frequency of updates to data is relatively low compared to reads


### SQL Server

For SQL Server the transaction log is used (via [dm_db_log_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-stats-transact-sql)
) if the current user has the `VIEW SERVER STATE` permission.

If `VIEW SERVER STATE` is not allowed then a combination of [Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/track-data-changes-sql-server) and/or [Row Versioning](https://learn.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql) is used.

Give the above certain kinds of operations will be detected:

|             | Transaction Log | Change Tracking | Row Versioning | Change Tracking<br>and Row Versioning |
|-------------|:---------------:|:---------------:|:--------------:|:----------------------------------:|
| Insert      |        ✅      |        ✅       |        ✅     |                  ✅                |
| Update      |        ✅      |        ✅       |        ✅     |                  ✅                |
| Hard Delete |        ✅      |        ✅       |        ❌     |                  ✅                |
| Soft Delete |        ✅      |        ✅       |        ✅     |                  ✅                |
| Truncate    |        ✅      |        ❌       |        ❌     |                  ❌                |



### Postgres

Postgres required [track_commit_timestamp](https://www.postgresql.org/docs/17/runtime-config-replication.html#GUC-TRACK-COMMIT-TIMESTAMP) to be enabled. This can be done using `ALTER SYSTEM SET track_commit_timestamp to "on"` and then restarting the Postgres service


## 304 Not Modified Flow

```mermaid
graph TD
    Request
    CalculateEtag[Calculate current ETag<br/>based on timestamp<br/>from web assembly and SQL]
    IfNoneMatch{Has<br/>If-None-Match<br/>header?}
    EtagMatch{Current<br/>Etag matches<br/>If-None-Match?}
    AddETag[Add current ETag<br/>to Response headers]
    304[Respond with<br/>304 Not-Modified]
    Request --> CalculateEtag
    CalculateEtag --> IfNoneMatch
    IfNoneMatch -->|Yes| EtagMatch
    IfNoneMatch -->|No| AddETag
    EtagMatch -->|No| AddETag
    EtagMatch -->|Yes| 304
```


## ETag calculation logic

The ETag is calculated from a combination several parts


### AssemblyWriteTime

The last write time of the web entry point assembly

<!-- snippet: AssemblyWriteTime -->
<a id='snippet-AssemblyWriteTime'></a>
```cs
var webAssemblyLocation = Assembly.GetEntryAssembly()!.Location;
AssemblyWriteTime = File.GetLastWriteTime(webAssemblyLocation).Ticks.ToString();
```
<sup><a href='/src/Delta/DeltaExtensions_Shared.cs#L38-L43' title='Snippet source file'>snippet source</a> | <a href='#snippet-AssemblyWriteTime' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### SQL timestamp


#### SQL Server


##### `VIEW SERVER STATE` permission

Transaction log is used via [dm_db_log_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-stats-transact-sql).

```sql
select log_end_lsn
from sys.dm_db_log_stats(db_id())
```


##### No `VIEW SERVER STATE` permission

A combination of [change_tracking_current_version](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/change-tracking-current-version-transact-sql) (if tracking is enabled) and [@@DBTS (row version timestamp)](https://learn.microsoft.com/en-us/sql/t-sql/functions/dbts-transact-sql)

```sql
declare @changeTracking bigint = change_tracking_current_version();
declare @timeStamp bigint = convert(bigint, @@dbts);

if (@changeTracking is null)
  select cast(@timeStamp as varchar)
else
  select cast(@timeStamp as varchar) + '-' + cast(@changeTracking as varchar)
```


#### Postgres

```sql
select pg_last_committed_xact();
```


### Suffix

An optional string suffix that is dynamically calculated at runtime based on the current `HttpContext`.

<!-- snippet: Suffix -->
<a id='snippet-Suffix'></a>
```cs
var app = builder.Build();
app.UseDelta(suffix: httpContext => "MySuffix");
```
<sup><a href='/src/DeltaTests/Usage.cs#L9-L14' title='Snippet source file'>snippet source</a> | <a href='#snippet-Suffix' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Combining the above

<!-- snippet: BuildEtag -->
<a id='snippet-BuildEtag'></a>
```cs
internal static string BuildEtag(string timeStamp, string? suffix)
{
    if (suffix == null)
    {
        return $"\"{AssemblyWriteTime}-{timeStamp}\"";
    }

    return $"\"{AssemblyWriteTime}-{timeStamp}-{suffix}\"";
}
```
<sup><a href='/src/Delta/DeltaExtensions_Shared.cs#L130-L142' title='Snippet source file'>snippet source</a> | <a href='#snippet-BuildEtag' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


## NuGet

Delta is shipped as two nugets:

 * [Delta](https://nuget.org/packages/Delta/): Delivers functionality using SqlConnection and SqlTransaction.
 * [Delta.EF](https://nuget.org/packages/Delta.EF/): Delivers functionality using [SQL Server EF Database Provider](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/?tabs=dotnet-core-cli).

Only one of the above should be used.


## Usage


### SQL Server DB Schema

Example SQL schema:

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


### Postgres DB Schema

Example SQL schema:

<!-- snippet: PostgresSchema -->
<a id='snippet-PostgresSchema'></a>
```cs
create table IF NOT EXISTS public."Companies"
(
    "Id" uuid not null
        constraint "PK_Companies"
            primary key,
    "Content" text
);

alter table public."Companies"
    owner to postgres;

create table IF NOT EXISTS public."Employees"
(
    "Id" uuid not null
        constraint "PK_Employees"
            primary key,
    "CompanyId" uuid not null
        constraint "FK_Employees_Companies_CompanyId"
            references public."Companies"
            on delete cascade,
    "Content"   text,
    "Age"       integer not null
);

alter table public."Employees"
    owner to postgres;

create index IF NOT EXISTS "IX_Employees_CompanyId"
    on public."Employees" ("CompanyId");
```
<sup><a href='/src/WebApplicationPostgres/PostgresDbBuilder.cs#L9-L39' title='Snippet source file'>snippet source</a> | <a href='#snippet-PostgresSchema' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Add to WebApplicationBuilder


#### SQL Server

<!-- snippet: UseDeltaSqlServer -->
<a id='snippet-UseDeltaSqlServer'></a>
```cs
var builder = WebApplication.CreateBuilder();
builder.Services.AddScoped(_ => new SqlConnection(connectionString));
var app = builder.Build();
app.UseDelta();
```
<sup><a href='/src/WebApplicationSqlServer/Program.cs#L10-L17' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaSqlServer' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### PostgreSQL

<!-- snippet: UseDeltaPostgres -->
<a id='snippet-UseDeltaPostgres'></a>
```cs
var builder = WebApplication.CreateBuilder();
builder.Services.AddScoped(_ => new NpgsqlConnection(connectionString));
var app = builder.Build();
app.UseDelta();
```
<sup><a href='/src/WebApplicationPostgres/Program.cs#L5-L12' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaPostgres' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Add to a Route Group

To add to a specific [Route Group](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-handlers#route-groups):

<!-- snippet: UseDeltaMapGroup -->
<a id='snippet-UseDeltaMapGroup'></a>
```cs
app.MapGroup("/group")
    .UseDelta()
    .MapGet("/", () => "Hello Group!");
```
<sup><a href='/src/WebApplicationSqlServer/Program.cs#L63-L69' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaMapGroup' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### ShouldExecute

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


### Custom Connection discovery

By default, Delta uses `HttpContext.RequestServices` to discover the SqlConnection and SqlTransaction:

<!-- snippet: DiscoverConnection -->
<a id='snippet-DiscoverConnection'></a>
```cs
static (Type sqlConnection, Type transaction) FindConnectionType()
{
    var sqlConnection = Type.GetType("Microsoft.Data.SqlClient.SqlConnection, Microsoft.Data.SqlClient");
    if (sqlConnection != null)
    {
        var transaction = sqlConnection.Assembly.GetType("Microsoft.Data.SqlClient.SqlTransaction")!;
        return (sqlConnection, transaction);
    }

    var npgsqlConnection = Type.GetType("Npgsql.NpgsqlConnection, Npgsql");
    if (npgsqlConnection != null)
    {
        var transaction = npgsqlConnection.Assembly.GetType("Npgsql.NpgsqlTransaction")!;
        return (npgsqlConnection, transaction);
    }

    throw new("Could not find connection type. Tried Microsoft.Data.SqlClient.SqlConnection and Npgsql.NpgsqlTransaction");
}

static Connection DiscoverConnection(HttpContext httpContext)
{
    var (connectionType, transactionType) = FindConnectionType();
    var provider = httpContext.RequestServices;
    var connection = (DbConnection) provider.GetRequiredService(connectionType);
    var transaction = (DbTransaction?) provider.GetService(transactionType);
    return new(connection, transaction);
}
```
<sup><a href='/src/Delta/DeltaExtensions_ConnectionDiscovery.cs#L5-L35' title='Snippet source file'>snippet source</a> | <a href='#snippet-DiscoverConnection' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

To use custom connection discovery:

<!-- snippet: CustomDiscoveryConnection -->
<a id='snippet-CustomDiscoveryConnection'></a>
```cs
var application = webApplicationBuilder.Build();
application.UseDelta(
    getConnection: httpContext => httpContext.RequestServices.GetRequiredService<SqlConnection>());
```
<sup><a href='/src/DeltaTests/Usage.cs#L341-L347' title='Snippet source file'>snippet source</a> | <a href='#snippet-CustomDiscoveryConnection' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

To use custom connection and transaction discovery:

<!-- snippet: CustomDiscoveryConnectionAndTransaction -->
<a id='snippet-CustomDiscoveryConnectionAndTransaction'></a>
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
<sup><a href='/src/DeltaTests/Usage.cs#L352-L364' title='Snippet source file'>snippet source</a> | <a href='#snippet-CustomDiscoveryConnectionAndTransaction' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


## EF Usage


### SqlServer DbContext using RowVersion

Enable row versioning in Entity Framework

<!-- snippet: SampleSqlServerDbContext -->
<a id='snippet-SampleSqlServerDbContext'></a>
```cs
public class SampleDbContext(DbContextOptions options) :
    DbContext(options)
{
    public DbSet<Employee> Employees { get; set; } = null!;
    public DbSet<Company> Companies { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder builder)
    {
        var company = builder.Entity<Company>();
        company.HasKey(_ => _.Id);
        company
            .HasMany(_ => _.Employees)
            .WithOne(_ => _.Company)
            .IsRequired();
        company
            .Property(_ => _.RowVersion)
            .IsRowVersion()
            .HasConversion<byte[]>();

        var employee = builder.Entity<Employee>();
        employee.HasKey(_ => _.Id);
        employee
            .Property(_ => _.RowVersion)
            .IsRowVersion()
            .HasConversion<byte[]>();
    }
}
```
<sup><a href='/src/WebApplicationSqlServerEF/DataContext/SampleDbContext.cs#L1-L31' title='Snippet source file'>snippet source</a> | <a href='#snippet-SampleSqlServerDbContext' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Postgres DbContext

Enable row versioning in Entity Framework

<!-- snippet: SamplePostgresDbContext -->
<a id='snippet-SamplePostgresDbContext'></a>
```cs
public class SampleDbContext(DbContextOptions options) :
    DbContext(options)
{
    public DbSet<Employee> Employees { get; set; } = null!;
    public DbSet<Company> Companies { get; set; } = null!;
    protected override void OnModelCreating(ModelBuilder builder)
    {
        var company = builder.Entity<Company>();
        company.HasKey(_ => _.Id);
        company
            .HasMany(_ => _.Employees)
            .WithOne(_ => _.Company)
            .IsRequired();

        var employee = builder.Entity<Employee>();
        employee.HasKey(_ => _.Id);
    }
}
```
<sup><a href='/src/WebApplicationPostgresEF/DataContext/SampleDbContext.cs#L1-L22' title='Snippet source file'>snippet source</a> | <a href='#snippet-SamplePostgresDbContext' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Add to WebApplicationBuilder


#### SQL Server

<!-- snippet: UseDeltaSQLServerEF -->
<a id='snippet-UseDeltaSQLServerEF'></a>
```cs
var builder = WebApplication.CreateBuilder();
builder.Services.AddSqlServer<SampleDbContext>(connectionString);
var app = builder.Build();
app.UseDelta<SampleDbContext>();
```
<sup><a href='/src/WebApplicationSqlServerEF/Program.cs#L9-L16' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaSQLServerEF' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### Postgres

<!-- snippet: UseDeltaPostgresEF -->
<a id='snippet-UseDeltaPostgresEF'></a>
```cs
var builder = WebApplication.CreateBuilder();
builder.Services.AddDbContext<SampleDbContext>(
    _ => _.UseNpgsql(connectionString));
var app = builder.Build();
app.UseDelta<SampleDbContext>();
```
<sup><a href='/src/WebApplicationPostgresEF/Program.cs#L5-L13' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaPostgresEF' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### Add to a Route Group

To add to a specific [Route Group](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-handlers#route-groups):

<!-- snippet: UseDeltaMapGroupEF -->
<a id='snippet-UseDeltaMapGroupEF'></a>
```cs
app.MapGroup("/group")
    .UseDelta<SampleDbContext>()
    .MapGet("/", () => "Hello Group!");
```
<sup><a href='/src/WebApplicationPostgresEF/Program.cs#L46-L52' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaMapGroupEF' title='Start of snippet'>anchor</a></sup>
<a id='snippet-UseDeltaMapGroupEF-1'></a>
```cs
app.MapGroup("/group")
    .UseDelta<SampleDbContext>()
    .MapGet("/", () => "Hello Group!");
```
<sup><a href='/src/WebApplicationSqlServerEF/Program.cs#L45-L51' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseDeltaMapGroupEF-1' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


### ShouldExecute

Optionally control what requests Delta is executed on.

<!-- snippet: ShouldExecuteEF -->
<a id='snippet-ShouldExecuteEF'></a>
```cs
var app = builder.Build();
app.UseDelta<SampleDbContext>(
    shouldExecute: httpContext =>
    {
        var path = httpContext.Request.Path.ToString();
        return path.Contains("match");
    });
```
<sup><a href='/src/Delta.EFTests/Usage.cs#L16-L26' title='Snippet source file'>snippet source</a> | <a href='#snippet-ShouldExecuteEF' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


## UseResponseDiagnostics

Response diagnostics is an opt-in feature that includes extra log information in the response headers.

Enable by setting UseResponseDiagnostics to true at startup:

<!-- snippet: UseResponseDiagnostics -->
<a id='snippet-UseResponseDiagnostics'></a>
```cs
DeltaExtensions.UseResponseDiagnostics = true;
```
<sup><a href='/src/DeltaTests/ModuleInitializer.cs#L6-L10' title='Snippet source file'>snippet source</a> | <a href='#snippet-UseResponseDiagnostics' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->

Response diagnostics headers are prefixed with `Delta-`.

Example Response header when the Request has not `If-None-Match` header.

<img src="/src/Delta-No304.png">


## Delta.SqlServer

A set of helper methods for working with [SQL Server Change Tracking](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/track-data-changes-sql-server) and [SQL Server Row Versioning](https://learn.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql)

Nuget: [Delta.SqlServer](https://www.nuget.org/packages/Delta.SqlServer)


### GetLastTimeStamp


#### For a `SqlConnection`:

<!-- snippet: GetLastTimeStampSqlConnection -->
<a id='snippet-GetLastTimeStampSqlConnection'></a>
```cs
var timeStamp = await sqlConnection.GetLastTimeStamp();
```
<sup><a href='/src/DeltaTests/Usage.cs#L193-L197' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetLastTimeStampSqlConnection' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


#### For a `DbContext`:

<!-- snippet: GetLastTimeStampEF -->
<a id='snippet-GetLastTimeStampEF'></a>
```cs
var timeStamp = await dbContext.GetLastTimeStamp();
```
<sup><a href='/src/Delta.EFTests/Usage.cs#L41-L45' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetLastTimeStampEF' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->


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
<sup><a href='/src/DeltaTests/Usage.cs#L229-L237' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetDatabasesWithTracking' title='Start of snippet'>anchor</a></sup>
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
<sup><a href='/src/DeltaTests/Usage.cs#L255-L263' title='Snippet source file'>snippet source</a> | <a href='#snippet-GetTrackedTables' title='Start of snippet'>anchor</a></sup>
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
<sup><a href='/src/DeltaTests/Usage.cs#L330-L334' title='Snippet source file'>snippet source</a> | <a href='#snippet-IsTrackingEnabled' title='Start of snippet'>anchor</a></sup>
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
<sup><a href='/src/DeltaTests/Usage.cs#L324-L328' title='Snippet source file'>snippet source</a> | <a href='#snippet-EnableTracking' title='Start of snippet'>anchor</a></sup>
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
<sup><a href='/src/DeltaTests/Usage.cs#L309-L313' title='Snippet source file'>snippet source</a> | <a href='#snippet-DisableTracking' title='Start of snippet'>anchor</a></sup>
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
<sup><a href='/src/DeltaTests/Usage.cs#L249-L253' title='Snippet source file'>snippet source</a> | <a href='#snippet-SetTrackedTables' title='Start of snippet'>anchor</a></sup>
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


## Verifying behavior

The behavior of Delta can be verified as follows:

 * Open a page in the site
 * Open the browser developer tools
 * Change to the Network tab
 * Refresh the page.

Cached responses will show as 304 in the `Status`:

<img src="/src/network.png">

In the headers `if-none-match` will show in the request and `etag` will show in the response:

<img src="/src/network-details.png">


### Ensure cache is not disabled

If disable cache is checked, the browser will not send the `if-none-match` header. This will effectively cause a cache miss server side, and the full server pipeline will execute.

<img src="/src/disable-cache.png">


### Certificates and Chromium

Chromium, and hence the Chrome and Edge browsers, are very sensitive to certificate problems when determining if an item should be cached. Specifically, if a request is done dynamically (type: xhr) and the server is using a self-signed certificate, then the browser will not send the `if-none-match` header. [Reference]( https://issues.chromium.org/issues/40666473). If self-signed certificates are required during development in lower environment, then use FireFox to test the caching behavior. 


## Programmatic client usage

Delta is primarily designed to support web browsers as a client. All web browsers have the necessary 304 and caching functionally required.

In the scenario where web apis (that support using 304) are being consumed using .net as a client, consider using one of the below extensions to cache responses.

 * [Replicant](https://github.com/SimonCropp/Replicant)
 * [Tavis.HttpCache](https://github.com/tavis-software/Tavis.HttpCache)
 * [CacheCow](https://github.com/aliostad/CacheCow)
 * [Monkey Cache](https://github.com/jamesmontemagno/monkey-cache)


## Icon

[Estuary](https://thenounproject.com/term/estuary/1847616/) designed by [Daan](https://thenounproject.com/Asphaleia/) from [The Noun Project](https://thenounproject.com).

