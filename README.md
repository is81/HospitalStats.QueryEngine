# HospitalStats.QueryEngine

**让 .NET 直连 Oracle 11g US7ASCII —— 告别中文乱码与 ROWNUM 分页噩梦**

[![License](https://img.shields.io/badge/license-AGPL%20v3-blue)](LICENSE)
[![NuGet](https://img.shields.io/badge/nuget-v1.0.0-teal)](https://www.nuget.org/)
[![.NET](https://img.shields.io/badge/.NET-8.0-purple)](https://dotnet.microsoft.com/)

纯本地 Oracle 查询引擎。处理老 HIS 数据库的 US7ASCII 中文编码、ROWNUM 分页、动态 SQL 生成。**零外网依赖，适合医院内网隔离环境。**

```csharp
// 10 行代码，直连 Oracle 11g US7ASCII
var request = new QueryEngineRequest {
    ConnectionString = "Data Source=ORCL;...",
    CharSetOverride  = "gbk",                    // ← 一行解决中文乱码
    MainTable = new() { TableName = "OUTP_BILL_ITEMS", SchemaName = "HOSPITAL", Alias = "A" },
    Fields = { new() { ColumnName = "ITEM_NAME", Alias = "项目" } },
    Filters = { new() { Id = 1, ColumnName = "ITEM_NAME", Operator = "LIKE", DefaultValue = "%阿莫西林%" } },
    Page = 1, PageSize = 50
};
var result = await engine.ExecuteAsync(request);
// result.Total  result.Rows  result.ElapsedMs
```

---

## 为什么需要它

| 痛点 | 引擎方案 |
|------|---------|
| 🔤 US7ASCII 传输中文变 `?` | 自动 RAWTOHEX 包装 + hex 解码，对调用方透明 |
| 📟 Oracle 10g 无 OFFSET | 内置 ROWNUM 三层子查询分页 |
| 🧩 动态 SQL 拼接地狱 | 12 种操作符 × 多表 JOIN × 聚合，参数化零拼接 |
| 🔒 医院内网不能连外网 | HMAC-SHA256 离线 License，零外发请求 |

## 安装

```bash
# 本地 NuGet 源（不需要外网）
dotnet nuget add source /path/to/nupkgs --name local
dotnet add package HospitalStats.QueryEngine
```

无 ASP.NET / EF Core / Redis 依赖，仅需 Dapper + ODP.NET。

## 使用

```csharp
// 1. License 激活（商业用户）
EngineLicense.InitializeOffline("your-license-key");

// 2. DI 注册
services.AddSingleton<IQueryEngine, QueryEngine>();

// 3. 执行查询
var result = await engine.ExecuteAsync(request);

// 4. 导出 Excel
var bytes = await engine.ExportExcelAsync(request);
```

[完整文档 →](https://github.com/is81/HospitalStats/blob/master/docs/%E6%9F%A5%E8%AF%A2%E5%BC%95%E6%93%8E%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95.md)

## 筛选操作符

`EQ` `NE` `GT` `GTE` `LT` `LTE` `LIKE` `NOT LIKE` `IN` `NOT IN` `BETWEEN` `NOT BETWEEN`

支持 `NOT LIKE::%关键词%` 格式覆盖默认操作符。上下文筛选器自动从 JWT Claims 注入，用户不可见。

## 双协议

| | AGPL v3 | 商业 License |
|------|---------|-------------|
| 费用 | 免费 | 付费（联系获取报价） |
| 闭源商用 | ❌ 产品必须开源 | ✅ 无限制 |
| 技术支持 | 社区 Issue | ✅ 邮件优先 |
| 获取方式 | `git clone` 本仓库 | 联系 is81@qq.com |

## API

| 方法 | 说明 |
|------|------|
| `ExecuteAsync(request)` | 执行查询，返回分页结果 + 耗时 |
| `ExportExcelAsync(request)` | 查询并导出 .xlsx |
| `GetDistinctValuesAsync(request)` | 获取列去重值（下拉框数据） |

## 技术规格

| 项目 | 值 |
|------|-----|
| 大小 | 61 KB DLL / 36 KB NuGet |
| 依赖 | Dapper + ODP.NET + ClosedXML + ILogger |
| Oracle 兼容 | 10g / 11g / 12c / 19c |
| 字符集 | US7ASCII / AL32UTF8 / ZHS16GBK |
| 最大结果集 | 50,000 行（可配置） |
| 网络 | **不需要** |

## 协议

[GNU Affero General Public License v3.0](LICENSE) — 开源项目免费使用，闭源商用需购买 License。

Copyright © 2026 is81. Contact: is81@qq.com
