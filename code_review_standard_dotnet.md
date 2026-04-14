# Code Review 標準規範（.NET / C#）

本規範收錄 .NET / C# 生態特有的常見錯誤，適用於以 C# 為主的後端服務 review。

## A1. Async / Await 陷阱

| 等級 | 描述 |
|------|------|
| **Critical** | 使用 `async void`（非 event handler 場景）：未捕捉的 exception 會直接 crash process |
| **Critical** | 阻塞 async 程式碼（`.Result`、`.GetAwaiter().GetResult()`）：在 ASP.NET Core 環境下會造成 thread pool starvation 與 deadlock |

## A2. 資源管理

| 等級 | 描述 |
|------|------|
| **Major** | `IDisposable` 物件（stream、DB connection、`HttpResponseMessage`）未使用 `using` 確保釋放，造成資源洩漏 |
| **Major** | 直接 `new HttpClient()` 而非透過 `IHttpClientFactory`，造成 socket exhaustion |

## A3. EF Core

| 等級 | 描述 |
|------|------|
| **Critical** | 多個相關 DB 寫入未使用 `IDbContextTransaction` 或 `TransactionScope` 包覆 |
| **Major** | 迴圈內的查詢可用 `Include()` 或批次查詢合併，但未優化 |
| **Major** | 分頁查詢缺少 `Skip`/`Take` |
| **Minor** | 唯讀查詢未標示 `AsNoTracking()`，造成不必要的 change tracking 開銷 |

## A4. 韌性框架

| 等級 | 描述 |
|------|------|
| **Major** | 對外部服務的呼叫未使用 Polly 實作 retry 與 circuit breaker |

## A5. 字串與文化

| 等級 | 描述 |
|------|------|
| **Major** | `string.Compare` 未加 `StringComparison` 參數，在不同 locale 環境行為不一致 |
| **Major** | 字串僅檢查 `null`，未使用 `string.IsNullOrWhiteSpace` 處理空字串與純空白 |
| **Major** | 迴圈內用 `string +=` 拼接大量字串，應改用 `StringBuilder` |

## A6. 集合與 LINQ

| 等級 | 描述 |
|------|------|
| **Minor** | `IEnumerable<T>` 參數在方法內部多次 enumerate，若呼叫方傳入 lazy sequence 會產生非預期行為，應在方法入口 `.ToList()` 固定 |
| **Minor** | `LINQ` 查詢在同一資料集上多次 enumerate，應先 `.ToList()` 固定結果 |
| **Minor** | Collection property 公開為 `List<T>` 而非 `IReadOnlyList<T>`，外部可隨意修改內部狀態 |

## A7. 時間

| 等級 | 描述 |
|------|------|
| **Critical** | 使用 `DateTime.Now` 而非 `DateTimeOffset.UtcNow`，在跨時區或 server 時區變更時產生資料錯誤 |

## A8. 配置與 DI

| 等級 | 描述 |
|------|------|
| **Major** | 直接讀 `ConfigurationManager` 或 hardcode 環境相關設定，而非透過 `IOptions<T>` 注入，導致設定無法測試與替換 |
| **Major** | Singleton service 內持有 Scoped 依賴（如 `DbContext`），造成 DbContext 永不釋放、多個 request 共用同一 context（.NET 會拋 `InvalidOperationException`） |
| **Minor** | 無狀態、建立成本低的 service 註冊為 Transient，應改為 Singleton 減少不必要的 allocation |

## A9. 並發

| 等級 | 描述 |
|------|------|
| **Major** | `ConcurrentDictionary.GetOrAdd` 傳入有 side effect 的 factory（如建立 DB 連線）， factory 可能被執行多次 |

## A10. 邊界條件

| 等級 | 描述 |
|------|------|
| **Critical** | 對集合直接呼叫 `.First()`、`.Single()` 或 `[0]` 而未先確認有元素 |
| **Minor** | 查詢缺少 `Take()` 上限保護 |

## A11. 套件安全

| 等級 | 描述 |
|------|------|
| **Info** | 建議定期執行 `dotnet list package --vulnerable` 掃描已知 CVE |
| **Minor** | 新增 NuGet 套件未審查 license、維護狀態與已知 CVE |
