# Code Review 標準規範（.NET / C#）

本規範收錄 .NET / C# 生態特有的常見錯誤，適用於以 C# 為主的後端服務 review。嚴重等級定義請參照通用規範。

---

## 一、Async / Await 陷阱

### Critical
- `async void`（非 event handler 場景）：未捕捉的 exception 會直接 crash process
- 阻塞 async 程式碼（`.Result`、`.GetAwaiter().GetResult()`）：在 thread pool 下會造成 starvation；傳統 ASP.NET（System.Web）環境另有 SynchronizationContext deadlock 風險，ASP.NET Core 已無此問題但仍應避免

---

## 二、資源管理

### Major
- `IDisposable` 物件（stream、DB connection、`HttpResponseMessage`）未使用 `using` 確保釋放，造成資源洩漏
- 直接 `new HttpClient()` 而非透過 `IHttpClientFactory`，造成 socket exhaustion

---

## 三、EF Core

### Critical
- 多個相關 DB 寫入未使用 `IDbContextTransaction` 或 `TransactionScope` 包覆

### Major
- 迴圈內的查詢可用 `Include()` 或批次查詢合併，但未優化
- 分頁查詢缺少 `Skip`/`Take`

### Minor
- 唯讀查詢未標示 `AsNoTracking()`，造成不必要的 change tracking 開銷

---

## 四、韌性框架

### Major
- 對外部服務的呼叫未使用 Polly 實作 retry 與 circuit breaker

---

## 五、字串與文化

### Major
- `string.Compare` 未加 `StringComparison` 參數，在不同 locale 環境行為不一致
- 字串僅檢查 `null`，未使用 `string.IsNullOrWhiteSpace` 處理空字串與純空白
- 迴圈內用 `string +=` 拼接大量字串，應改用 `StringBuilder`

---

## 六、集合與 LINQ

### Major
- `IEnumerable<T>` 參數在方法內部多次 enumerate，若呼叫方傳入 lazy sequence（如 DB query、有副作用的序列）會重複執行查詢或副作用，應在方法入口 `.ToList()` 固定

### Minor
- LINQ 查詢在同一資料集上多次 enumerate，應先 `.ToList()` 固定結果
- Collection property 公開為 `List<T>` 而非 `IReadOnlyList<T>`，外部可隨意修改內部狀態

---

## 七、時間

### Critical
- 使用 `DateTime.Now` 而非 `DateTimeOffset.UtcNow`，在跨時區或 server 時區變更時產生資料錯誤

---

## 八、配置與 DI

### Major
- 直接讀 `ConfigurationManager` 或 hardcode 環境相關設定，而非透過 `IOptions<T>` 注入，導致設定無法測試與替換
- Singleton service 內持有 Scoped 依賴（如 `DbContext`），造成 DbContext 永不釋放、多個 request 共用同一 context（.NET 會拋 `InvalidOperationException`）

### Minor
- 無狀態、建立成本低且不依賴 Scoped 服務的 service 註冊為 Transient，應改為 Singleton 減少不必要的 allocation

---

## 九、並發

### Major
- `ConcurrentDictionary.GetOrAdd` 傳入有 side effect 的 factory（如建立 DB 連線），factory 可能被執行多次

---

## 十、邊界條件

### Critical
- 對集合直接呼叫 `.First()`、`.Single()` 或 `[0]` 而未先確認有元素

### Minor
- 查詢缺少 `Take()` 上限保護

---

## 十一、套件安全

### Info
- 建議定期執行 `dotnet list package --vulnerable` 掃描已知 CVE
