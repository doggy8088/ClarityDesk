# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

**ClarityDesk** 是一個基於 ASP.NET Core 8.0 的問題追蹤管理系統，使用 Razor Pages 架構，整合 LINE Login OAuth 認證。系統主要用於管理客戶問題回報單，支援多部門協作處理。

**技術棧**:
- .NET 8.0 (LTS)
- ASP.NET Core Razor Pages
- Entity Framework Core 8.0 + SQL Server
- LINE OAuth 2.0 認證
- EPPlus (Excel 匯出)

## 常用開發命令

### 建置與執行
```bash
# 還原套件
dotnet restore

# 建置專案
dotnet build

# 執行應用程式 (開發模式)
dotnet run

# 執行應用程式並監聽檔案變更 (熱重載)
dotnet watch run

# 發佈 Release 版本
dotnet publish -c Release -o ./publish
```

### 資料庫操作 (Entity Framework Core)
```bash
# 新增資料庫遷移 (Migration)
dotnet ef migrations add <MigrationName>

# 更新資料庫至最新版本
dotnet ef database update

# 回復至特定 Migration
dotnet ef database update <PreviousMigrationName>

# 列出所有 Migrations
dotnet ef migrations list

# 產生 SQL 腳本 (不執行)
dotnet ef migrations script

# 移除最後一個 Migration (尚未套用至資料庫時)
dotnet ef migrations remove
```

### 測試
```bash
# 執行所有測試
dotnet test

# 執行特定測試專案
dotnet test Tests/ClarityDesk.UnitTests/
dotnet test Tests/ClarityDesk.IntegrationTests/

# 執行測試並顯示詳細輸出
dotnet test --verbosity detailed
```

### 程式碼檢查
```bash
# 檢查程式碼格式
dotnet format --verify-no-changes

# 自動修正程式碼格式
dotnet format
```

## 高層架構設計

### 分層架構 (Layered Architecture)

```
┌─────────────────────────────────────────────────┐
│  Presentation Layer (Pages/)                    │
│  - Razor Pages (.cshtml + PageModel)            │
│  - ViewModels                                    │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│  Application Layer (Services/)                  │
│  - IssueReportService (回報單管理)              │
│  - AuthenticationService (LINE 登入)            │
│  - DepartmentService (部門管理)                 │
│  - UserManagementService (使用者管理)           │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│  Domain Layer (Models/)                         │
│  - Entities (資料實體)                          │
│  - DTOs (資料傳輸物件)                          │
│  - Enums (列舉類型)                             │
│  - Extensions (擴充方法: Entity ↔ DTO 轉換)    │
└─────────────────────┬───────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│  Infrastructure Layer                           │
│  - Data/ (EF Core DbContext + Configurations)   │
│  - Infrastructure/ (Authentication, Middleware) │
└─────────────────────────────────────────────────┘
```

### 核心實體關聯

```
User (使用者)
  ├─ 1:N → IssueReport (作為 AssignedUser)
  ├─ 1:N → IssueReport (作為 LastModifiedBy)
  └─ N:M → Department (透過 DepartmentUser)

IssueReport (回報單)
  ├─ N:1 → User (指派人員)
  ├─ N:1 → User (最後修改人)
  └─ N:M → Department (透過 DepartmentAssignment)

Department (部門)
  ├─ N:M → IssueReport (透過 DepartmentAssignment)
  └─ N:M → User (透過 DepartmentUser)
```

**關鍵設計模式**:
- **Service Layer Pattern**: 所有業務邏輯封裝在 Services/Interfaces
- **DTO Pattern**: 使用 DTOs 在層間傳遞資料，避免直接暴露實體
- **Extension Methods**: Entity ↔ DTO 轉換透過 `Models/Extensions/` 實現
- **Fluent API Configuration**: 使用 `IEntityTypeConfiguration<T>` 配置實體關係 (在 `Data/Configurations/`)

## 關鍵架構決策與設計

### 1. LINE Login OAuth 認證流程

**配置位置**: `Program.cs:50-124`

認證流程:
1. 使用者訪問需要登入的頁面 → 導向 `/Account/Login`
2. 點擊 LINE 登入按鈕 → 導向 LINE OAuth 授權頁面
3. 使用者授權後，LINE 回調至 `/signin-line`
4. `OnCreatingTicket` 事件處理器:
   - 呼叫 LINE Profile API 取得使用者資料
   - 呼叫 `AuthenticationService.LoginOrRegisterWithLineAsync()` 建立/更新本地使用者
   - 寫入 Claims: UserId, Role, Name, Picture
5. 建立 Cookie Authentication (永久會話，365 天)

**授權策略**:
- **全域授權**: `/Issues/*` 需要登入
- **Admin Policy**: `/Admin/*` 需要 Admin 角色
- **匿名頁面**: Login, AccessDenied, Error, Index, Privacy

**重要**: LINE OAuth 憑證必須在 `appsettings.json` 中正確配置:
```json
"LineLogin": {
  "ChannelId": "YOUR_LINE_CHANNEL_ID",
  "ChannelSecret": "YOUR_LINE_CHANNEL_SECRET",
  "CallbackPath": "/signin-line"
}
```

### 2. Entity Framework Core 設計模式

**DbContext**: `Data/ApplicationDbContext.cs` (116 行)

**Fluent API Configurations** (推薦新增實體時遵循):
```csharp
// 範例: Data/Configurations/IssueReportConfiguration.cs
public class IssueReportConfiguration : IEntityTypeConfiguration<IssueReport>
{
    public void Configure(EntityTypeBuilder<IssueReport> builder)
    {
        // Primary Key
        builder.HasKey(i => i.Id);

        // Required Fields + Max Length
        builder.Property(i => i.Title).IsRequired().HasMaxLength(200);

        // Foreign Keys
        builder.HasOne(i => i.AssignedUser)
            .WithMany()
            .HasForeignKey(i => i.AssignedUserId)
            .OnDelete(DeleteBehavior.Restrict);

        // Indexes (效能優化)
        builder.HasIndex(i => i.Status);
        builder.HasIndex(i => new { i.Status, i.Priority }); // 複合索引
    }
}
```

**自動時間戳記**: `ApplicationDbContext` 覆寫 `SaveChangesAsync()` 自動更新 `CreatedAt` / `UpdatedAt`:
```csharp
// 新增實體時自動設定 CreatedAt
if (entry.State == EntityState.Added)
    entity.CreatedAt = DateTime.UtcNow;

// 修改實體時自動更新 UpdatedAt
entity.UpdatedAt = DateTime.UtcNow;
```

**注意**: 所有新實體類別必須在 `ApplicationDbContext.OnModelCreating()` 中套用 Configuration:
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new YourNewEntityConfiguration());
}
```

### 3. Service Layer 設計原則

**服務介面定義** (位於 `Services/Interfaces/`):
```csharp
public interface IIssueReportService
{
    Task<int> CreateIssueReportAsync(CreateIssueReportDto dto, int createdByUserId);
    Task<bool> UpdateIssueReportAsync(int id, UpdateIssueReportDto dto, int modifiedByUserId);
    Task<PagedResult<IssueReportDto>> GetIssueReportsAsync(IssueFilterDto filter);
    // ... 其他方法
}
```

**依賴注入註冊** (在 `Program.cs`):
```csharp
builder.Services.AddScoped<IIssueReportService, IssueReportService>();
builder.Services.AddScoped<IAuthenticationService, AuthenticationService>();
builder.Services.AddScoped<IUserManagementService, UserManagementService>();
builder.Services.AddScoped<IDepartmentService, DepartmentService>();
```

**服務層職責**:
- 業務邏輯驗證與處理
- 協調多個實體的操作 (例如: 建立回報單 + 部門指派)
- DTO ↔ Entity 轉換 (透過 Extension Methods)
- 事務管理 (EF Core 自動處理)

**新增服務時的步驟**:
1. 在 `Services/Interfaces/` 建立介面
2. 在 `Services/` 實作服務類別
3. 在 `Program.cs` 註冊依賴注入
4. 在 PageModel 建構函式注入使用

### 4. DTO 與 Extension Methods 模式

**DTO 定義** (位於 `Models/DTOs/`):
```csharp
// CreateDto: 用於建立實體 (不含 Id)
public class CreateIssueReportDto { ... }

// UpdateDto: 用於更新實體 (不含唯讀欄位)
public class UpdateIssueReportDto { ... }

// Dto: 用於讀取與顯示 (含完整資訊)
public class IssueReportDto { ... }
```

**Extension Methods** (位於 `Models/Extensions/`):
```csharp
public static class IssueReportExtensions
{
    // Entity → DTO (讀取)
    public static IssueReportDto ToDto(this IssueReport entity) { ... }

    // CreateDto → Entity (建立)
    public static IssueReport ToEntity(this CreateIssueReportDto dto) { ... }

    // UpdateDto → Entity (更新)
    public static bool UpdateFromDto(this IssueReport entity, UpdateIssueReportDto dto) { ... }
}
```

**優點**:
- 避免直接暴露實體給前端 (安全性)
- 控制序列化/反序列化的欄位 (避免 Over-posting)
- 方便 API 版本控制與演進

### 5. 效能優化策略

**記憶體快取** (Program.cs:143):
```csharp
builder.Services.AddMemoryCache();

// 使用範例 (IssueReportService.cs)
private const string CacheKeyStatistics = "issue_statistics";
private const int CacheExpirationMinutes = 5;

var statistics = await _cache.GetOrCreateAsync(CacheKeyStatistics, async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(CacheExpirationMinutes);
    return await CalculateStatisticsAsync();
});
```

**回應壓縮** (Brotli + Gzip):
```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});
```

**靜態檔案快取** (365 天):
```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        const int durationInSeconds = 60 * 60 * 24 * 365;
        ctx.Context.Response.Headers.Append("Cache-Control",
            $"public,max-age={durationInSeconds}");
    }
});
```

**資料庫查詢優化**:
- 使用 `.AsNoTracking()` 進行唯讀查詢
- 索引優化: 單欄位索引 + 複合索引 (Status, Priority, RecordDate)
- 使用 `.Include()` 預先載入關聯資料 (避免 N+1 問題)

**範例**:
```csharp
var issues = await _context.IssueReports
    .Include(i => i.AssignedUser)
    .Include(i => i.DepartmentAssignments)
        .ThenInclude(da => da.Department)
    .Where(i => i.Status == IssueStatus.Pending)
    .AsNoTracking() // 唯讀查詢
    .ToListAsync();
```

### 6. 審計追蹤 (Audit Trail)

所有主要實體包含審計欄位:
- `CreatedAt` (DateTime): 建立時間
- `UpdatedAt` (DateTime): 最後更新時間
- `LastModifiedByUserId` (int?, IssueReport 專用): 最後修改人

**自動更新**: `ApplicationDbContext.SaveChangesAsync()` 自動處理
**手動設定**: 服務層方法接受 `createdByUserId` / `modifiedByUserId` 參數

## 重要檔案與位置

### 核心配置
- `Program.cs` (227 行) - 應用程式啟動與服務配置
- `appsettings.json` - 資料庫連線字串、LINE Login 憑證

### 資料層
- `Data/ApplicationDbContext.cs` - EF Core DbContext
- `Data/ApplicationDbContextSeed.cs` - 種子資料初始化
- `Data/Configurations/*.cs` - Fluent API 實體配置

### 業務邏輯層
- `Services/IssueReportService.cs` (422 行) - 回報單管理
- `Services/AuthenticationService.cs` (119 行) - LINE 登入認證
- `Services/DepartmentService.cs` (344 行) - 部門管理
- `Services/UserManagementService.cs` (129 行) - 使用者管理

### 領域模型層
- `Models/Entities/*.cs` - 資料實體
- `Models/DTOs/*.cs` - 資料傳輸物件
- `Models/Enums/*.cs` - 列舉類型 (UserRole, IssueStatus, PriorityLevel)
- `Models/Extensions/*.cs` - Entity ↔ DTO 轉換擴充方法

### 基礎設施層
- `Infrastructure/Authentication/LineAuthenticationHandler.cs` - LINE OAuth 處理器
- `Infrastructure/Middleware/ExceptionHandlingMiddleware.cs` - 全域例外處理

### 資料庫
- `Migrations/*.cs` - EF Core 資料庫遷移
- `database/MSSQL-結構與範例資料建立檔.sql` - SQL 初始化腳本

## 開發工作流程

### 新增功能的標準流程

1. **定義領域模型** (如果需要新實體):
   - 在 `Models/Entities/` 建立實體類別
   - 在 `Data/Configurations/` 建立 Fluent API 配置
   - 在 `ApplicationDbContext.OnModelCreating()` 套用配置
   - 執行 `dotnet ef migrations add AddNewEntity`
   - 執行 `dotnet ef database update`

2. **建立 DTOs**:
   - 在 `Models/DTOs/` 建立 CreateDto, UpdateDto, Dto
   - 在 `Models/Extensions/` 建立 Extension Methods (ToDto, ToEntity, UpdateFromDto)

3. **實作服務層**:
   - 在 `Services/Interfaces/` 定義服務介面
   - 在 `Services/` 實作服務類別
   - 在 `Program.cs` 註冊依賴注入

4. **實作 Razor Pages**:
   - 在 `Pages/` 建立 `.cshtml` (View) + `.cshtml.cs` (PageModel)
   - 在 PageModel 建構函式注入服務
   - 在 `Program.cs` 設定授權策略 (如果需要)

5. **測試**:
   - 在 `Tests/ClarityDesk.UnitTests/` 建立單元測試
   - 在 `Tests/ClarityDesk.IntegrationTests/` 建立整合測試

### 修改現有功能

1. **定位相關檔案**:
   - 從 Razor Page 開始 (Pages/)
   - 追蹤至 PageModel 注入的 Service
   - 檢視 Service 使用的 Entity 與 DTO

2. **修改順序** (由內而外):
   - Entity (如果需要) → Migration → Update Database
   - DTO 與 Extension Methods
   - Service 層業務邏輯
   - PageModel 與 Razor Page

### 除錯與日誌

**開發環境除錯**:
- `appsettings.json` 已啟用 `DetailedErrors: true`
- `ExceptionHandlingMiddleware` 在開發環境顯示詳細錯誤訊息

**日誌記錄**:
```csharp
// Service 層注入 ILogger
private readonly ILogger<IssueReportService> _logger;

_logger.LogInformation("Creating issue report for user {UserId}", userId);
_logger.LogError(ex, "Failed to update issue {IssueId}", issueId);
```

## 常見開發情境

### 修改資料表結構

```bash
# 1. 修改 Entity 類別
# 2. 更新對應的 Configuration (如果需要)
# 3. 新增 Migration
dotnet ef migrations add UpdateIssueReportSchema

# 4. 檢視產生的 Migration 程式碼
# 5. 套用至資料庫
dotnet ef database update

# 如果 Migration 有問題,可以回復並重新產生
dotnet ef database update PreviousMigrationName
dotnet ef migrations remove
```

### 新增使用者角色

當前角色定義在 `Models/Enums/UserRole.cs`:
```csharp
public enum UserRole
{
    User = 0,   // 一般使用者
    Admin = 1   // 系統管理員
}
```

如需新增角色:
1. 修改 `UserRole` enum
2. 更新 `Program.cs` 授權策略
3. 更新種子資料 (如果需要)
4. 新增 Migration: `dotnet ef migrations add AddNewUserRole`

### 調整 LINE Login 設定

LINE Login 憑證必須在 LINE Developers Console 取得:
1. 前往 https://developers.line.biz/
2. 建立 Provider 與 LINE Login Channel
3. 設定 Callback URL: `https://your-domain.com/signin-line`
4. 取得 Channel ID 與 Channel Secret
5. 更新 `appsettings.json`:
   ```json
   "LineLogin": {
     "ChannelId": "YOUR_CHANNEL_ID",
     "ChannelSecret": "YOUR_CHANNEL_SECRET",
     "CallbackPath": "/signin-line"
   }
   ```

### Excel 匯出功能

使用 EPPlus 套件實現 (範例: `Pages/Issues/Index.cshtml.cs`):
```csharp
using OfficeOpenXml;

public async Task<IActionResult> OnPostExportAsync()
{
    var issues = await GetFilteredIssuesAsync();

    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Issues");

    // 設定標題列
    worksheet.Cells[1, 1].Value = "編號";
    worksheet.Cells[1, 2].Value = "標題";
    // ...

    // 填入資料
    for (int i = 0; i < issues.Count; i++)
    {
        worksheet.Cells[i + 2, 1].Value = issues[i].Id;
        worksheet.Cells[i + 2, 2].Value = issues[i].Title;
        // ...
    }

    // 自動調整欄寬
    worksheet.Cells[worksheet.Dimension.Address].AutoFitColumns();

    var stream = new MemoryStream(package.GetAsByteArray());
    return File(stream, "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                $"Issues_{DateTime.Now:yyyyMMdd}.xlsx");
}
```

## 部署注意事項

### 首次部署檢查清單

- [ ] 安裝 .NET 8.0 SDK/Runtime
- [ ] 安裝 SQL Server 2019+ (或使用 Azure SQL Database)
- [ ] 建立資料庫與登入帳號
- [ ] 設定資料庫連線字串 (appsettings.json 或環境變數)
- [ ] 設定 LINE Login 憑證 (appsettings.json 或環境變數)
- [ ] 執行資料庫 Migration: `dotnet ef database update`
- [ ] 確認種子資料已初始化 (系統管理員與預設部門)
- [ ] 設定 HTTPS 憑證 (生產環境)
- [ ] 設定防火牆規則 (允許 SQL Server 連線)

### 環境變數 (生產環境建議)

```bash
# 連線字串
ConnectionStrings__DefaultConnection="Server=...;Database=ClarityDesk;..."

# LINE Login
LineLogin__ChannelId="YOUR_CHANNEL_ID"
LineLogin__ChannelSecret="YOUR_CHANNEL_SECRET"
LineLogin__CallbackPath="/signin-line"

# ASP.NET Core 環境
ASPNETCORE_ENVIRONMENT="Production"
ASPNETCORE_URLS="http://+:5000;https://+:5001"
```

### IIS 部署

參考 `web.config` (已包含在專案中):
```xml
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" />
    </handlers>
    <aspNetCore processPath="dotnet"
                arguments=".\ClarityDesk.dll"
                stdoutLogEnabled="false"
                hostingModel="inprocess" />
  </system.webServer>
</configuration>
```

### 資料庫備份

```sql
-- 完整備份
BACKUP DATABASE ClarityDesk
TO DISK = 'C:\Backups\ClarityDesk_Full.bak'
WITH FORMAT, INIT, NAME = 'Full Backup of ClarityDesk';

-- 差異備份
BACKUP DATABASE ClarityDesk
TO DISK = 'C:\Backups\ClarityDesk_Diff.bak'
WITH DIFFERENTIAL, FORMAT, INIT;
```

## 專案特定慣例

### 命名慣例
- **Entities**: 單數名詞 (User, Department, IssueReport)
- **DbSet**: 複數名詞 (Users, Departments, IssueReports)
- **Services**: `I{Entity}Service` 介面, `{Entity}Service` 實作
- **DTOs**: `{Entity}Dto`, `Create{Entity}Dto`, `Update{Entity}Dto`
- **Extension Methods**: `{Entity}Extensions`

### 程式碼風格
- 使用 C# 12 新語法 (檔案範圍命名空間、全域 using)
- Nullable Reference Types 已啟用
- 使用 `var` 進行區域變數宣告 (型別明確時)
- Async/Await 所有非同步方法 (命名以 `Async` 結尾)

### Git 工作流程
- Main Branch: `main` (生產環境)
- Development Branch: `develop` (開發環境)
- Feature Branches: `feature/{feature-name}`
- Hotfix Branches: `hotfix/{issue-description}`

### Commit 訊息格式
專案包含 PowerShell 腳本輔助 commit (scripts/commit-changes.ps1)

建議格式:
- `feat: 新增使用者管理功能`
- `fix: 修正回報單狀態更新問題`
- `refactor: 重構部門服務層邏輯`
- `docs: 更新 CLAUDE.md 文件`
- `chore: 更新 EF Core 至 8.0.1`

## 故障排除

### 常見問題

**Q: 執行 `dotnet run` 時出現資料庫連線錯誤**
A: 檢查 `appsettings.json` 的 `ConnectionStrings:DefaultConnection` 設定，確認 SQL Server 已啟動且可連線。

**Q: LINE Login 無法運作**
A:
1. 確認 `appsettings.json` 的 LINE Login 憑證正確
2. 檢查 LINE Developers Console 的 Callback URL 設定是否正確
3. 確認應用程式 URL 與 Callback URL 的 domain 一致

**Q: Migration 失敗**
A:
```bash
# 檢視詳細錯誤訊息
dotnet ef database update --verbose

# 回復至上一個 Migration
dotnet ef database update PreviousMigrationName

# 產生 SQL 腳本手動檢視
dotnet ef migrations script
```

**Q: Excel 匯出失敗 (EPPlus License Exception)**
A: EPPlus 7.0+ 需要設定授權上下文 (已在 Program.cs 設定):
```csharp
ExcelPackage.LicenseContext = LicenseContext.NonCommercial;
```

**Q: 靜態檔案 (CSS/JS) 無法載入**
A: 確認 `app.UseStaticFiles()` 在 `Program.cs` 的 middleware pipeline 中正確設定。

### 效能問題診斷

**查詢 N+1 問題**:
- 啟用 EF Core 查詢日誌: `options.UseSqlServer(...).EnableSensitiveDataLogging()`
- 使用 `.Include()` 與 `.ThenInclude()` 預先載入關聯資料

**記憶體洩漏**:
- 確認所有 `IDisposable` 資源 (DbContext, HttpClient) 正確 Dispose
- 使用 `using` 陳述式或 `using` 宣告

**資料庫鎖定**:
- 檢查長時間執行的事務
- 考慮使用 `.AsNoTracking()` 進行唯讀查詢
- 調整 SQL Server 隔離層級 (如果需要)

## 授權

本專案採用 MIT 授權條款 (詳見 LICENSE 檔案)。
