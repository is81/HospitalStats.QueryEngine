# HospitalStats.QueryEngine

**Connect .NET to Oracle 11g US7ASCII — No More Mojibake or ROWNUM Pagination Pain**

[![License](https://img.shields.io/badge/license-AGPL%20v3-blue)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-8.0-purple)](https://dotnet.microsoft.com/)
[![NuGet](https://img.shields.io/badge/nuget-v1.0.0-teal)](https://www.nuget.org/)

A fully offline Oracle query execution engine for legacy HIS databases. Handles US7ASCII Chinese text encoding, ROWNUM pagination for Oracle 10g, and dynamic SQL generation. **Zero external network dependency** — designed for air-gapped hospital environments. 61 KB DLL, four dependencies.

```csharp
// 10 lines, straight to Oracle 11g US7ASCII
var request = new QueryEngineRequest {
    ConnectionString = "Data Source=ORCL;...",
    CharSetOverride  = "gbk",                    // ← one line fixes Chinese encoding
    MainTable = new() { TableName = "OUTP_BILL_ITEMS", SchemaName = "HOSPITAL", Alias = "A" },
    Fields   = { new() { ColumnName = "ITEM_NAME", Alias = "Item" } },
    Filters  = { new() { Id = 1, ColumnName = "ITEM_NAME", Operator = "LIKE", DefaultValue = "%amoxicillin%" } },
    Page = 1, PageSize = 50
};
var result = await engine.ExecuteAsync(request);
// result.Total  result.Rows  result.ElapsedMs
```

> [中文文档](README.md)

---

## Why This Exists

Tens of thousands of hospitals in China still run Oracle 10g/11g with US7ASCII character sets. Developers integrating with legacy HIS databases face three problems daily:

| Problem | Engine Solution |
|---------|----------------|
| 🔤 Chinese text becomes `?` during US7ASCII transport | Automatic RAWTOHEX wrapping + hex decode, transparent to caller |
| 📟 Oracle 10g/11g has no `OFFSET/FETCH` | Built-in ROWNUM three-layer subquery pagination |
| 🧩 Dynamic filter × multi-table JOIN × aggregation | 12 operators, parameterized queries, zero SQL injection |
| 🔒 Hospital network must be air-gapped | HMAC‑SHA256 offline license validation, zero outbound requests |

## Installation

Distributed as a NuGet package. No nuget.org required — just place the `.nupkg` file in a local directory.

```bash
# Local source install (no internet needed)
dotnet nuget add source /path/to/packages --name local
dotnet add package HospitalStats.QueryEngine
```

Dependencies: Dapper · Oracle.ManagedDataAccess.Core · ClosedXML · Microsoft.Extensions.Logging.Abstractions  
**Zero** ASP.NET · **Zero** EF Core · **Zero** Redis · **Zero** HTTP client

## Quick Start

```csharp
// 1. License activation (commercial users)
EngineLicense.InitializeOffline("your-license-key", "your-hmac-secret");

// 2. DI registration
services.AddSingleton<IQueryEngine, QueryEngine>();

// 3. Execute query
var result = await engine.ExecuteAsync(request);

// 4. Export to Excel
var bytes = await engine.ExportExcelAsync(request);
File.WriteAllBytes("result.xlsx", bytes);

// 5. Get distinct values for dropdown
var options = await engine.GetDistinctValuesAsync(new DistinctValuesRequest {
    ConnectionString = "...", TableName = "DEPT_DICT", ColumnName = "DEPT_NAME"
});
```

Full documentation (Chinese): [`docs/查询引擎使用方法.md`](docs/查询引擎使用方法.md)

## Filter Operators

| Operator | Meaning | Operator | Meaning |
|----------|---------|----------|---------|
| `EQ` | Equals | `NOT LIKE` | Not contains |
| `NE` | Not equals | `IN` | In list |
| `GT` | Greater than | `NOT IN` | Not in list |
| `GTE` | Greater or equal | `BETWEEN` | Range |
| `LT` | Less than | `NOT BETWEEN` | Not in range |
| `LTE` | Less or equal | `LIKE` | Contains |

Overrides: `NOT LIKE::%keyword%` replaces the default operator at runtime. Date types auto-wrapped with `TO_DATE()`.

## US7ASCII Handling

Set `CharSetOverride = "gbk"` and the engine performs four automatic steps:

1. **SQL preprocessing** — Chinese literals `'内科'` → `RAWTOHEX(HEXTORAW('...'))`
2. **Column wrapping** — String output columns wrapped with `RAWTOHEX(UTL_RAW.CAST_TO_RAW("COL"))`
3. **Parameter encoding** — Chinese filter values hex-encoded before binding, preventing ODP.NET corruption
4. **Result decoding** — hex → GBK/GB2312/GB18030 auto-restore, LIKE wildcards preserved

## Context Filters

Filters marked `IsContextFilter = true` are auto-injected from `ContextValues`. Users never see or override them:

```csharp
Filters = { new() {
    Id = 100, ColumnName = "DEPT_NAME",
    IsContextFilter = true, ContextKey = "DeptName"
}},
ContextValues = new() { ["DeptName"] = "Cardiology" }
// Generates: WHERE "DEPT_NAME" = 'Cardiology' (invisible to end users)
```

## Tech Specs

| Item | Value |
|------|-------|
| Size | 61 KB DLL / 36 KB NuGet |
| Runtime | .NET 8 |
| Oracle | 10g / 11g / 12c / 19c |
| Charsets | US7ASCII / AL32UTF8 / ZHS16GBK / WE8ISO8859P1 |
| Max rows | 50,000 (configurable) |
| Pagination | ROWNUM three-layer subquery |
| Concurrency | Thread-safe (connection per request) |
| Network | **Not required** |

## Dual License

| | AGPL v3 | Commercial License |
|------|---------|-------------------|
| Cost | Free | Paid |
| Source available | ✅ GitHub | ✅ Same AGPL source |
| Closed-source use | ❌ Your product must be open-source too | ✅ Unrestricted |
| Copyleft waiver | ❌ | ✅ |
| IP indemnification | ❌ | ✅ |
| Support | Community Issues | Priority email |
| For | Open-source projects, learning | HIS vendors, proprietary products |
| Get it | `git clone` this repo | Contact is81@qq.com |

**TL;DR**: If your product is also open-source, AGPL is free. For closed-source commercial use, buy a license.

## API

| Method | Description |
|--------|-------------|
| `ExecuteAsync(request, ct)` | Execute query, returns paginated results with row count and timing |
| `ExportExcelAsync(request, ct)` | Query and export .xlsx (auto-bold headers, auto-fit columns) |
| `GetDistinctValuesAsync(request, ct)` | Get distinct column values (for filter dropdowns) |

## Pipeline

```
Your Code
  → IQueryEngine.ExecuteAsync(QueryEngineRequest)
    → OracleConnection + Dapper
      → Auto hex-encode (US7ASCII)
      → Dynamic SQL (COUNT + ROWNUM pagination)
      → Parameterized binding (injection-safe)
    ← Hex-decode back to readable text
  ← QueryEngineResult { Rows, Columns, Total, ElapsedMs }
```

## Related Projects

- [Community Edition (MIT)](https://github.com/is81/HospitalStats) — Full web admin dashboard
- [Enterprise Edition (Commercial)](https://github.com/is81/HospitalStats) — Pivot tables, scheduled reports, DRG analysis, SSO

## License

This repository is licensed under the [GNU Affero General Public License v3.0](LICENSE). In short: **free for open-source, paid license required for closed-source commercial use.**

Copyright © 2026 is81. Contact: is81@qq.com
