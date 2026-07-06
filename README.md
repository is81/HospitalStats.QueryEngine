# HospitalStats.QueryEngine

**让 .NET 直连 Oracle 11g US7ASCII —— 告别中文乱码与 ROWNUM 分页噩梦**

[![License](https://img.shields.io/badge/license-AGPL%20v3-blue)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-8.0-purple)](https://dotnet.microsoft.com/)
[![NuGet](https://img.shields.io/badge/nuget-v1.0.0-teal)](https://www.nuget.org/)

纯本地 Oracle 查询执行引擎。专为老 HIS 数据库设计，处理 US7ASCII 中文编码、Oracle 10g ROWNUM 分页、动态 SQL 生成。**零外网依赖**，适合医院内网隔离环境。36 KB 轻量，四个依赖包。

```csharp
// 10 行代码，直连 Oracle 11g US7ASCII
var request = new QueryEngineRequest {
    ConnectionString = "Data Source=ORCL;...",
    CharSetOverride  = "gbk",                    // ← 一行解决中文乱码
    MainTable = new() { TableName = "OUTP_BILL_ITEMS", SchemaName = "HOSPITAL", Alias = "A" },
    Fields   = { new() { ColumnName = "ITEM_NAME", Alias = "项目" } },
    Filters  = { new() { Id = 1, ColumnName = "ITEM_NAME", Operator = "LIKE", DefaultValue = "%阿莫西林%" } },
    Page = 1, PageSize = 50
};
var result = await engine.ExecuteAsync(request);
// result.Total  result.Rows  result.ElapsedMs
```

---

## 为什么需要它

中国数万家医院仍在运行 Oracle 10g/11g + US7ASCII 字符集。开发者在对接老 HIS 数据库时，每天面对三个问题：

| 痛点 | 引擎方案 |
|------|---------|
| 🔤 US7ASCII 传输中文变 `?`，患者姓名、药品名称损坏 | 自动 RAWTOHEX 包裹 + hex 解码还原，对调用方透明 |
| 📟 Oracle 10g/11g 不支持 `OFFSET/FETCH` | 内置 ROWNUM 三层嵌套子查询，传 `page` 和 `pageSize` 即可 |
| 🧩 可变筛选 × 多表 JOIN × 聚合拼装 SQL | 12 种操作符，参数化查询，零 SQL 注入风险 |
| 🔒 医院内网必须物理隔离，不能连外网 | HMAC‑SHA256 离线 License 验证，零外发网络请求 |

## 安装

引擎以 NuGet 包分发。不需要发布到 nuget.org，把 `.nupkg` 文件放到本地目录即可。

```bash
# 本地源安装（不需要外网）
dotnet nuget add source /path/to/packages --name local
dotnet add package HospitalStats.QueryEngine
```

依赖：Dapper · Oracle.ManagedDataAccess.Core · ClosedXML · Microsoft.Extensions.Logging.Abstractions  
**零** ASP.NET · **零** EF Core · **零** Redis · **零** HTTP 客户端

## 快速开始

```csharp
// 1. License 激活（商业用户）
EngineLicense.InitializeOffline("your-license-key", "your-hmac-secret");

// 2. DI 注册
services.AddSingleton<IQueryEngine, QueryEngine>();

// 3. 执行查询
var result = await engine.ExecuteAsync(request);

// 4. 导出 Excel
var bytes = await engine.ExportExcelAsync(request);
File.WriteAllBytes("查询结果.xlsx", bytes);

// 5. 获取筛选下拉选项
var options = await engine.GetDistinctValuesAsync(new DistinctValuesRequest {
    ConnectionString = "...", TableName = "DEPT_DICT", ColumnName = "DEPT_NAME"
});
```

完整文档见 [`docs/查询引擎使用方法.md`](docs/查询引擎使用方法.md)

## 筛选操作符

| 操作符 | 说明 | 操作符 | 说明 |
|--------|------|--------|------|
| `EQ` | 等于 | `NOT LIKE` | 模糊排除 |
| `NE` | 不等于 | `IN` | 包含于 |
| `GT` | 大于 | `NOT IN` | 不包含于 |
| `GTE` | 大于等于 | `BETWEEN` | 范围 |
| `LT` | 小于 | `NOT BETWEEN` | 范围排除 |
| `LTE` | 小于等于 | `LIKE` | 模糊匹配 |

支持 `NOT LIKE::%关键词%` 格式动态覆盖默认操作符。日期类型自动 `TO_DATE()` 包装。

## US7ASCII 处理

设置 `CharSetOverride = "gbk"` 后，引擎自动执行四步处理，全程对调用方透明：

1. **SQL 预处理** — 中文字面量 `'内科'` → `RAWTOHEX(HEXTORAW('C4DABFC6'))`
2. **列包装** — 字符串输出列包裹 `RAWTOHEX(UTL_RAW.CAST_TO_RAW("COL"))`
3. **参数编码** — 中文筛选值 hex 编码后绑定，防止 ODP.NET 传输损坏
4. **结果解码** — hex → GBK / GB2312 / GB18030 自动还原，LIKE 通配符保留

## 上下文筛选器

标记 `IsContextFilter = true` 的筛选条件，值从 `ContextValues` 自动注入，用户不可见、不可覆盖：

```csharp
Filters = { new() {
    Id = 100, ColumnName = "DEPT_NAME",
    IsContextFilter = true, ContextKey = "DeptName"
}},
ContextValues = new() { ["DeptName"] = "内科" }
// 生成: WHERE "DEPT_NAME" = '内科'（自动注入，用户看不到这个条件）
```

## 技术规格

| 项目 | 值 |
|------|-----|
| 大小 | 61 KB DLL / 36 KB NuGet |
| 运行时 | .NET 8 |
| Oracle 兼容 | 10g / 11g / 12c / 19c |
| 字符集 | US7ASCII / AL32UTF8 / ZHS16GBK / WE8ISO8859P1 |
| 最大结果集 | 50,000 行（可配置） |
| 分页方式 | ROWNUM 三层嵌套子查询 |
| 并发 | 线程安全（连接按请求创建） |
| 网络 | 不需要外网 |

## 双协议

医院信息科和 HIS 厂商的常见问题：**我能免费用到什么程度？**

| | AGPL v3 | 商业 License |
|------|---------|-------------|
| 费用 | 免费 | 付费 |
| 源码可见 | ✅ GitHub 公开 | ✅ 同 AGPL 源码 |
| 闭源商用 | ❌ 你的产品也必须开源 | ✅ 无限制 |
| 免 Copyleft 义务 | ❌ | ✅ |
| 知识产权免责 | ❌ | ✅ |
| 技术支持 | 社区 Issue | 邮件优先 |
| 适用对象 | 开源项目、个人学习 | HIS 厂商、闭源产品 |
| 获取方式 | `git clone` 本仓库 | 联系 is81@qq.com |

**简单理解**：如果你的产品也开源，AGPL 完全免费。如果闭源商用，买一个 License。

## API

| 方法 | 说明 |
|------|------|
| `ExecuteAsync(request, ct)` | 执行查询，返回分页结果、总行数、耗时 |
| `ExportExcelAsync(request, ct)` | 查询并导出 .xlsx（自动表头加粗、列宽自适应） |
| `GetDistinctValuesAsync(request, ct)` | 获取列去重值（筛选下拉框数据源） |

## 调用链

```
你的代码
  → IQueryEngine.ExecuteAsync(QueryEngineRequest)
    → OracleConnection + Dapper
      → 自动 hex 编码（US7ASCII）
      → 动态 SQL 生成（COUNT + ROWNUM 分页）
      → 参数化绑定（防注入）
    ← hex 解码还原中文
  ← QueryEngineResult { Rows, Columns, Total, ElapsedMs }
```

## 相关项目

- [社区版（MIT）](https://github.com/is81/HospitalStats) — 完整 Web 管理后台
- [企业版（商业）](https://github.com/is81/HospitalStats) — 交叉表、定时报告、DRG 分析、SSO

## 协议

本仓库以 [GNU Affero General Public License v3.0](LICENSE) 授权。简单说：**开源项目免费使用，闭源商用需要购买 License。**

Copyright © 2026 is81. 联系：is81@qq.com
