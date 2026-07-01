# HospitalStats.QueryEngine User Guide

> Last updated: 2026-07-01

Standalone Oracle query execution engine with zero ASP.NET dependencies. Handles US7ASCII Chinese text encoding, Oracle 10g ROWNUM pagination, and dynamic SQL generation.

**Core feature: fully offline operation.** License validation, Oracle queries, SQL generation — all performed locally, no external network required. Designed for air-gapped hospital environments.

---

## 1. Getting the Engine

The engine is distributed as a `.nupkg` file. **No nuget.org required. No internet needed.**

### Option 1: Local NuGet Source (Recommended)

Place the `.nupkg` file in a local directory and register it as a source:

```bash
# Put the package file in a local directory
mkdir C:\packages
copy HospitalStats.QueryEngine.1.0.0.nupkg C:\packages\

# Add local NuGet source
dotnet nuget add source C:\packages --name HospitalStats-Local

# Install
dotnet add package HospitalStats.QueryEngine
```

### Option 2: Copy DLL Directly

Copy `HospitalStats.QueryEngine.dll` into your project and add a manual reference.

### Option 3: Source Reference (AGPL v3)

```xml
<ProjectReference Include="..\HospitalStats.QueryEngine\HospitalStats.QueryEngine.csproj" />
```

### Publishing to nuget.org (Optional)

If you later want `dotnet add package` to work directly (still requires NuGet client internet access), you can publish to [nuget.org](https://nuget.org). This is entirely optional. Current v1.0.0 package:

```
F:\HospitalStats\deploy\nupkgs\HospitalStats.QueryEngine.1.0.0.nupkg
```

---

## 2. License Activation

### Offline Mode (Recommended — Zero Server Dependency, No Internet)

The engine includes a built-in HMAC-SHA256 offline validator. The license key itself carries signed authorization data (licensee name, expiry date, feature tier). The engine verifies the signature and checks expiry **entirely offline**.

```csharp
// Program.cs / Startup
using HospitalStats.QueryEngine;

// One-line init — pure local validation, no network. HMAC key provided by vendor.
EngineLicense.InitializeOffline("your-license-key", "your-hmac-secret");

// Register engine
builder.Services.AddSingleton<IQueryEngine, QueryEngine>();
```

The license key is a Base64 string (format: `payload.signature`). Send it to customers via email / WeChat / file — they paste it into their code.

### Online Mode (Optional — Custom Validation Server)

If you later need online validation (e.g., per-call billing), inject a custom callback:

```csharp
EngineLicense.Initialize(async (key) =>
{
    using var http = new HttpClient();
    var resp = await http.PostAsync(
        "https://your-server.com/api/license/validate",
        new StringContent($"{{\"key\":\"{key}\"}}"));
    return resp.IsSuccessStatusCode;
}, licenseKey);
```

### AGPL Mode (Source Use — No License Required)

When compiling from source, simply don't call `EngineLicense.Initialize()`. The engine silently skips validation (no-op).

---

## 3. Basic Usage

### 3.1 DI Registration

```csharp
builder.Services.AddSingleton<IQueryEngine, QueryEngine>();
```

### 3.2 Building a Query Request

```csharp
var request = new QueryEngineRequest
{
    // Oracle connection string (plaintext — caller handles decryption)
    ConnectionString = "User Id=scott;Password=tiger;Data Source=ORCL",

    // Set encoding for US7ASCII databases (e.g. "gbk")
    CharSetOverride = null,  // or "gbk"

    // Main table
    MainTable = new EngineTableDef
    {
        TableName = "OUTP_BILL_ITEMS",
        SchemaName = "HOSPITAL",
        Alias = "A"
    },

    // Output fields
    Fields = new List<EngineFieldDef>
    {
        new() { ColumnName = "ITEM_CODE",  Alias = "Code",   TableAlias = "A" },
        new() { ColumnName = "ITEM_NAME",  Alias = "Name",   TableAlias = "A", DataType = "VARCHAR2" },
        new() { ColumnName = "ITEM_PRICE", Alias = "Price",  TableAlias = "A", DataType = "NUMBER" }
    },

    // Filter conditions
    Filters = new List<EngineFilterDef>
    {
        new()
        {
            Id = 1,
            ColumnName = "ITEM_NAME",
            TableAlias = "A",
            DataType = "VARCHAR2",
            Operator = "LIKE",
            DefaultValue = "%amoxicillin%",
            SortOrder = 0
        }
    },

    // Joined tables
    Joins = new List<EngineJoinDef>
    {
        new()
        {
            JoinType = "LEFT",
            JoinTable = new EngineTableDef
            {
                TableName = "DEPT_DICT",
                SchemaName = "HOSPITAL",
                Alias = "B"
            },
            LeftColumnName = "DEPT_CODE",
            LeftTableAlias = "A",
            RightColumnName = "DEPT_CODE",
            RightTableAlias = "B",
            SortOrder = 0
        }
    },

    // Raw SQL (optional — overrides Fields/Joins/Filters)
    RawSql = null,

    // Sorting & grouping
    SortColumn = "A.ITEM_CODE",
    SortDirection = "ASC",
    GroupByColumn = null,

    // Pagination
    Page = 1,
    PageSize = 50,

    // Filter values (key matches FilterDef.Id)
    FilterValues = new Dictionary<string, string>
    {
        ["1"] = "%amoxicillin%"
    },

    // Context filter values (e.g., current user department)
    ContextValues = new Dictionary<string, string>
    {
        ["DeptName"] = "Cardiology",
        ["UserId"] = "42"
    },

    // Runtime options
    Options = new EngineOptions
    {
        QueryTimeoutSeconds = 120,
        MaxRowCount = 50000
    }
};
```

### 3.3 Executing a Query

```csharp
// Resolve IQueryEngine
var engine = serviceProvider.GetRequiredService<IQueryEngine>();

// Execute
var result = await engine.ExecuteAsync(request);

// Results
Console.WriteLine($"Total: {result.Total}");
Console.WriteLine($"Page:  {result.Page}/{Math.Ceiling(result.Total / (double)result.PageSize)}");
Console.WriteLine($"Time:  {result.ElapsedMs}ms");

foreach (var row in result.Rows)
{
    // row is Dictionary<string, object?>
    Console.WriteLine(string.Join(", ", row.Select(kv => $"{kv.Key}={kv.Value}")));
}

// result.Columns: column names in output order
// result.Rows: data rows
// result.Total: total rows (before pagination)
// result.ElapsedMs: execution time (milliseconds)
```

### 3.4 Exporting to Excel

```csharp
request.Page = 1;
request.PageSize = request.Options.MaxRowCount;  // export all

var bytes = await engine.ExportExcelAsync(request);
File.WriteAllBytes("export.xlsx", bytes);
```

### 3.5 Getting Dropdown Values

```csharp
var distinctRequest = new DistinctValuesRequest
{
    ConnectionString = "User Id=scott;Password=tiger;Data Source=ORCL",
    CharSetOverride = "gbk",
    SchemaName = "HOSPITAL",
    TableName = "OUTP_BILL_ITEMS",
    TableAlias = "A",
    ColumnName = "ITEM_NAME",
    DataType = "VARCHAR2"
};

var options = await engine.GetDistinctValuesAsync(distinctRequest);
// ["Amoxicillin", "Cefradine", "Ibuprofen", ...]
```

---

## 4. Filter Operators

| Operator | SQL Output | FilterValues Example |
|----------|-----------|---------------------|
| `EQ` | `col = :p` | `"amoxicillin"` |
| `NE` | `col != :p` | `"amoxicillin"` |
| `GT` | `col > :p` | `"100"` |
| `GTE` | `col >= :p` | `"100"` |
| `LT` | `col < :p` | `"2024-01-01"` |
| `LTE` | `col <= :p` | `"2024-01-01"` |
| `LIKE` | `col LIKE :p` | `"%amoxicillin%"` |
| `NOT LIKE` | `col NOT LIKE :p` | `"%test%"` |
| `IN` | `col IN (:p_0, :p_1, ...)` | `"A001,B002,C003"` |
| `NOT IN` | `col NOT IN (:p_0, ...)` | `"T1,T2"` |
| `BETWEEN` | `col BETWEEN :p_from AND :p_to` | `"100,500"` |
| `NOT BETWEEN` | `col NOT BETWEEN :p_from AND :p_to` | `"2024-01-01,2024-12-31"` |

**Operator prefix override**: use `::` in FilterValues to override the default operator:
```
"NOT LIKE::%test%"  → uses NOT LIKE instead of default
"GTE::2024-01-01"   → uses GTE instead of default
```

---

## 5. Context Filters

Context filters auto-inject values from `ContextValues`. Users never see or override them:

```csharp
Filters = new List<EngineFilterDef>
{
    new()
    {
        Id = 100,
        ColumnName = "DEPT_NAME",
        Operator = "EQ",
        IsContextFilter = true,
        ContextKey = "DeptName"    // matches key in ContextValues
    }
},

ContextValues = new Dictionary<string, string>
{
    ["DeptName"] = "Cardiology",   // auto-injected, user can't override
    ["UserId"] = "42"
}
```

Context filters with no value are silently skipped (no error).

---

## 6. US7ASCII Encoding

For US7ASCII Oracle databases, set `CharSetOverride`:

```csharp
CharSetOverride = "gbk"  // or "gb2312" / "gb18030"
```

The engine performs four automatic steps:

1. **Count SQL**: `HexEncodeInlineLiterals` converts Chinese literals to hex
2. **Data SQL**: `HexEncodeRawSqlColumns` wraps string output columns with `RAWTOHEX(UTL_RAW.CAST_TO_RAW(...))`
3. **Filter values**: `EncodeNonAsciiValue` hex-encodes Chinese parameter values to prevent transport corruption
4. **Result decoding**: auto-calls `DecodeHexString` / `ConvertEncoding` before returning results

### Encoding Utility Methods (Can Be Used Standalone)

```csharp
using HospitalStats.QueryEngine;

// Check if a type is string-like
EncodingHelper.IsStringType("VARCHAR2");  // true

// Check for non-ASCII characters
EncodingHelper.ContainsNonAscii("内科");  // true

// Decode hex string
EncodingHelper.DecodeHexString("C4DABFC6", "gbk");  // "内科"

// Fix garbled text
EncodingHelper.ConvertEncoding(garbleText, "gbk");

// Hex-encode Chinese literals in SQL
EncodingHelper.HexEncodeInlineLiterals(sql, "gbk");
```

---

## 7. Raw SQL Mode

When `RawSql` is set, the engine uses it directly instead of building from Fields/Joins/Filters:

```csharp
var request = new QueryEngineRequest
{
    ConnectionString = "...",
    RawSql = @"
        SELECT a.patient_id, a.name, b.dept_name
        FROM clinic_master a
        LEFT JOIN dept_dict b ON a.dept_code = b.dept_code
        WHERE a.visit_date >= '2024-01-01'
    ",
    // Fields can be empty (engine parses column names from RawSql)
    Fields = new List<EngineFieldDef>(),
    // Filters still apply (injected into the RawSql WHERE clause)
    Filters = new List<EngineFilterDef> { ... },
    Page = 1,
    PageSize = 50
};
```

Processing pipeline:
1. **Validate**: only SELECT / WITH queries allowed
2. **Inject filters**: `InjectWhereIntoRawSql` inserts user filters before GROUP BY / ORDER BY
3. **Hex encode** (US7ASCII): `HexEncodeRawSqlColumns` wraps string columns
4. **Paginate**: ROWNUM three-layer subquery
5. **Execute**: Dapper parameterized query

UNION queries are also supported — the engine calls `SplitUnionBranches` and injects filters per branch.

---

## 8. License Management

### Getting a License Key

Commercial license keys are issued by the vendor (is81@qq.com). Each key contains:
- Licensee name (licensedTo)
- Issue date (issuedAt)
- Expiry date (expiresAt)
- Feature tier (tier)

### Delivery

The license key and HMAC signing secret are sent to the customer via email. Configure once in code:

```csharp
EngineLicense.InitializeOffline(licenseKey, hmacSecret);
```

### Checking Status

```csharp
if (EngineLicense.IsLicensed)
    Console.WriteLine("Activated");
else
    Console.WriteLine("Not activated (AGPL mode)");
```

### Tamper Protection

- HMAC-SHA256 signature — any modification invalidates the key
- `CryptographicOperations.FixedTimeEquals` — timing attack resistant
- 7-day grace period — production continues if validation is unreachable

---

## 9. Complete Integration Examples

### .NET 8 Minimal API

```csharp
using HospitalStats.QueryEngine;

var builder = WebApplication.CreateBuilder(args);

// 1. Activate license
EngineLicense.InitializeOffline(builder.Configuration["License:Key"]!, builder.Configuration["License:HmacSecret"]!);

// 2. Register engine
builder.Services.AddSingleton<IQueryEngine, QueryEngine>();

var app = builder.Build();

// 3. Use engine
app.MapPost("/api/query", async (IQueryEngine engine, QueryEngineRequest request) =>
{
    var result = await engine.ExecuteAsync(request);
    return Results.Ok(result);
});

app.Run();
```

### Console Application

```csharp
using HospitalStats.QueryEngine;

// Initialize
EngineLicense.InitializeOffline(args[0], args[1]); // license key + hmac secret from command line

var engine = new QueryEngine(LoggerFactory.Create(b => b.AddConsole())
    .CreateLogger<QueryEngine>());

var request = new QueryEngineRequest
{
    ConnectionString = "User Id=scott;Password=tiger;Data Source=ORCL",
    MainTable = new EngineTableDef { TableName = "DEPT_DICT", SchemaName = "HOSPITAL", Alias = "D" },
    Fields = new List<EngineFieldDef>
    {
        new() { ColumnName = "DEPT_NAME", TableAlias = "D" }
    },
    Page = 1,
    PageSize = 10
};

var result = await engine.ExecuteAsync(request);
foreach (var row in result.Rows)
    Console.WriteLine(row["DEPT_NAME"]);
```

---

## 10. FAQ

### Q: Does the engine require internet access?

**No.** The engine is designed from the ground up for fully offline operation:
- License validation: local HMAC-SHA256 signature check, no network
- Oracle queries: direct connection to hospital intranet Oracle
- No telemetry, no tracking, no outbound network requests
- The `.nupkg` file can be distributed via email / WeChat / intranet share — no nuget.org needed

Suitable for physically isolated hospital intranet environments.

### Q: What's the relationship with nuget.org?

None. nuget.org is Microsoft's public NuGet package repository (analogous to npm). Publishing there enables `dotnet add package` to work directly. However, this requires internet access and is not required.

The current package is a `HospitalStats.QueryEngine.1.0.0.nupkg` file (36 KB) that can be copied directly to the customer's machine and installed from a local source. Publishing to nuget.org is entirely optional.

### Q: How does the AGPL license affect me?

If the product you build with this engine is also open-source under an AGPL-compatible license — free to use, no license key needed.

If you build a closed-source commercial product — you need a commercial license. The commercial license includes an IP indemnification clause, technical support, and priority bug fixes.

### Q: What happens when the license key expires?

The engine continues working for a 7-day grace period (preventing production outages from validation failures), then throws an exception prompting renewal. The grace period starts from the last successful validation.

### Q: Can I skip EngineLicense.Initialize?

Yes. Without calling `Initialize`, the engine works normally (no-op). This applies to AGPL source users.

### Q: How do I get a commercial license?

Contact is81@qq.com for pricing and a license key.

---

*Document version 1.0, for HospitalStats.QueryEngine v1.0.0*
