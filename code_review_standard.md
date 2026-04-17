# Code Review 標準規範（通用版）

## 嚴重等級定義

| 等級 | 說明 | Merge 條件 |
|------|------|-----------|
| **Critical** | 會造成 bug、資安漏洞、資料遺失、系統崩潰 | 必須修正，不得 merge |
| **Major** | 顯著影響可維護性、效能、可觀測性或工程紀律 | 原則上必須修正，或提出書面理由後由 reviewer 裁量放行 |
| **Minor** | 建議改善，不阻擋功能 | 可於後續 PR 修正，需開 follow-up issue |
| **Info** | 風格建議、學習資源、觀察備註 | 不要求回應 |

---

## 一、程式碼品質（可讀性、結構、抽象）

### Critical
- 類別同時承擔超過一個職責（SRP 明顯違反），導致改動一處影響無關功能

### Major
- 相同邏輯複製超過 **2 份**，未抽取為共用方法或服務（Copy-paste duplication）
- 業務邏輯寫在入口層（Controller / Handler / Route），應移至 Service / Domain 層
- 命名無法表達意圖（單字母變數、縮寫無法望文生義，排除迴圈 `i/j`）
- 不知何時該用 Design Pattern：多個 `if/else` 依類型分派不同行為未用 Strategy；物件建立邏輯複雜且散落各處未用 Factory；一個改動需修改多個不相關模組未用 Observer / Event；同一參數層層傳遞穿越多個方法未用 Context object 或 DI
- Magic number / magic string 直接寫在邏輯中（ID、語言代碼、欄位索引等）未宣告為具名常數
- 單一方法超過 **200 行**且包含多個業務邏輯分支（God Method），應拆分為具名子方法
- **Overengineering**：為目前不存在的需求加入抽象層、介面、泛型或 plugin 機制，實際只有一個實作且短期內不會擴充
- **Overengineering**：使用 design pattern（Strategy、Factory、Observer 等）解決一個簡單的 `if/else` 問題，增加理解成本而無實質收益
- **Overengineering**：Premature optimization——在沒有效能數據支撐的情況下引入複雜的 cache 策略、非同步佇列或分散式架構

### Minor
- 方法超過 **50 行**但邏輯尚可理解，建議拆分
- 過度拆分：方法只有 2–3 行且只被呼叫一次，拆出來反而增加跳讀負擔，應內聯
- 類別/方法命名使用動詞/名詞混亂（如 `UserHelper`、`DataManager`）
- 巢狀 `if` 超過 3 層，可用 early return 或策略模式簡化
- 迴圈內大量字串拼接使用低效方式（如重複 concatenation），應使用語言提供的高效字串建構工具

### Info
- 可考慮使用語言慣用語法（idioms）改善可讀性

---

## 二、工程紀律（Commit Message、版控習慣）

### Critical
- Commit 包含 **binary 檔案**（編譯產物、套件、IDE 設定等不應進版控的二進位檔）
- Commit 包含 **機密資訊**（API key、密碼、連線字串明文）

### Major
- Commit message 無法描述「做了什麼、為什麼」（如 `fix`、`update`、`wip`、`asdf`）
- 單一 commit 混合多個無關的變更（應拆為獨立 commit）
- 直接 push to `main` / `develop` 而未走 MR 流程

### Minor
- Commit message 未遵守專案約定格式（如缺少 issue 編號前綴）
- Branch 命名不符合規範（如 `fix-bug` 而非 `bugfix/PROJ-123-description`）
- 新增第三方套件未審查 license、維護狀態與已知 CVE

### Info
- 建議使用 Conventional Commits 格式（`feat:`, `fix:`, `refactor:` 等）

---

## 三、效能與正確性（N+1、Null Safety、Cache Strategy）

### Critical
- **N+1 Query**：在迴圈內逐次打 DB，未以批次查詢或預先載入合併，有分頁時每頁仍可能觸發大量查詢
- Null dereference 風險：直接存取可能為 null 的物件成員未做 null 檢查，且物件來源為外部輸入或 DB 查詢結果
- 同一查詢在同一 request 內重複執行兩次以上，結果未被快取重用

### Major
- 迴圈內的 DB 查詢可用批次查詢或預先載入（eager loading）合併，但未優化
- 可快取的熱點資料（記憶體或分散式快取）未做任何快取
- 分頁查詢缺少 offset/limit，回傳全表資料至記憶體後再過濾
- Null 安全不一致：同一方法內對相同來源物件混用安全存取與直接存取，隱含假設不一致

### Minor
- 頻繁呼叫的 API 回應未設定 HTTP cache header 或 ETag
- 唯讀查詢未標示為唯讀，造成不必要的追蹤或鎖定開銷

### Info
- 可評估是否需要 query result cache（如 Redis）以減少 DB 壓力

---

## 四、安全性

### Critical
- SQL 字串拼接（未使用參數化查詢），有 SQL Injection 風險
- 使用者輸入未經驗證直接用於檔案路徑、Shell 指令（Path Traversal / Command Injection）
- 敏感資料（密碼、token）以明文記錄於 log
- 授權驗證缺失（endpoint 未加權限檢查或驗證邏輯被跳過）

### Major
- 錯誤訊息將 stack trace 或內部架構細節回傳給用戶端
- CORS 設為 `*` 於非公開 API
- 密碼使用弱雜湊演算法（MD5、SHA1），未使用 bcrypt / Argon2 等慢雜湊

### Minor
- 未設定 HTTP Security Headers（`X-Content-Type-Options`、`X-Frame-Options` 等）
- Dependency 有已知 CVE 且有可用更新版本

### Info
- 建議定期執行套件漏洞掃描（各語言生態有對應工具）

---

## 五、可維護性

### Critical
- Dead code 留在 production（被 comment 掉的邏輯、永遠不會執行的分支），且無說明原因

### Major
- 錯誤靜默中斷：catch block 為空、`if null → return/break` 無任何 log 或向上拋出，導致呼叫方無從得知失敗
- 直接依賴具體實作而非抽象介面（直接建構具體類別而非透過依賴注入），影響測試與替換
- 公開的 Service / Repository 方法無任何文件說明，且邏輯非顯而易見

### Minor
- `TODO` / `FIXME` 留在 code 中超過一個 sprint 未處理且無對應 issue

---

## 六、可觀測性（Observability）

### Critical
- 新 endpoint 或關鍵業務邏輯異動未加入 metrics 埋點（latency、error rate、throughput），導致無法在 production 偵測異常

### Major
- Log 為純文字格式而非結構化（JSON），且缺少 **correlation ID**，無法跨服務追蹤單一 request
- 引入新的外部依賴呼叫（DB、第三方 API）未加任何 log，失敗時無從診斷

### Minor
- Log level 使用不當：預期內的錯誤（如 validation failure）記錄為 `ERROR`，造成 alert 噪音；或將除錯資訊記錄為 `INFO`，污染正式環境 log
- Exception 被 catch 後僅記錄 message，未記錄完整 stack trace

### Info
- 建議導入 OpenTelemetry 統一 tracing / metrics / logging 三支柱

---

## 七、可擴展性與韌性（Scalability & Resilience）

### Critical
- Fire-and-forget 非同步執行（如背景 task）未正確捕捉 exception，導致例外被靜默吞掉或 crash process
- 阻塞非同步執行：在非同步框架中同步等待非同步結果，可能造成 thread pool 耗盡或 deadlock

### Major
- 需共用的連線型資源（HTTP client、DB connection pool）每次使用都重新建立，造成 socket / connection 耗盡
- 對外部服務（HTTP API、message queue、DB）的呼叫沒有 retry 與 circuit breaker，單點失敗會造成 cascading outage
- 外部服務呼叫未設定 timeout，依賴方無回應時會無限等待，佔用執行緒或連線
- 外部資料（API response、DB 結果）假設格式一定符合預期，未處理缺欄位、型別不符或空回應的情況
- 需要明確釋放的資源（檔案 handle、DB connection、網路連線）未確保在所有路徑下釋放，造成資源洩漏

### Minor
- 查詢缺少筆數上限保護，資料量在 staging 環境下正常 but production 可能回傳百萬筆

### Info
- 長時間執行的操作建議支援 cancellation，以便 graceful shutdown

---

## 八、API 設計與版本控制（API Design & Versioning）

### Critical
- **Breaking Change 禁止未版本化**：既存對外 API 的欄位禁止直接刪除或更名；若需更動，必須透過版本號（Versioning）或保留舊欄位並標註 Deprecated，確保向下相容性，避免下游服務或 Mobile App 靜默崩潰

### Major
- **直接回傳內部 domain model / ORM entity** 作為 API response，導致 domain 與 API contract 強耦合，無法獨立演進；應使用專用 DTO
- **Chatty API 設計**：用戶端需發多次連續請求才能組出一個完整畫面，應考慮合併為單一端點或使用 BFF 模式
- **回傳過量資料**：response 包含用戶端不需要的欄位，造成不必要的序列化與網路傳輸開銷


---

## 九、並發與執行緒安全（Concurrency & Thread Safety）

### Critical
- 可變的共享狀態（全域或模組級別的變數）被多個並發執行緒/請求共寫，未加適當同步保護，造成 race condition
- 多個相關的 DB 寫入操作未包在同一 transaction 內，部分失敗會造成資料不一致

### Major
- 鎖（lock / mutex）的範圍包住 I/O 操作（DB call、HTTP call），造成執行緒阻塞，應縮小鎖的範圍
- 方法修改傳入的參數物件（mutation of input），呼叫方無從得知物件狀態已被改變
- 集合屬性公開為完全可寫型別，外部可隨意修改內部狀態，應改為唯讀視圖

### Singleton 誤用
- **Critical**：Singleton 服務持有 request-scoped 依賴（如 DB context），造成跨請求共用同一份狀態，導致資料污染
- **Critical**：手刻 Singleton（靜態 instance + lazy init）且非 thread-safe，concurrent 存取可能建出多份或讀取到未初始化狀態
- **Major**：有可變狀態的服務設計為 Singleton，多個請求並發寫入同一份狀態，造成 race condition
- **Major**：手刻 Singleton 而非透過 DI container 管理，導致無法注入 mock，測試困難
- **Minor**：無狀態、建立成本低的服務設計為每次建立新實例，應改為共用實例減少不必要的 allocation

---

## 十、例外設計與錯誤處理（Exception Design）

### Major
- 用 exception 做流程控制（以 try/catch 取代條件判斷）——exception 應保留給真正的例外情況
- 過度通用的 catch（如 `catch Exception` / `catch (err)`）掩蓋非預期錯誤，應捕捉最小範圍的具體型別

### Minor
- 拋出過於通用的 exception type（如直接 `throw new Error("message")`），呼叫方無法針對性處理，應定義有語意的自訂型別

---

## 十一、時間、文化與設定（Time, Culture & Configuration）

### Critical
- 使用本地時間而非 UTC 時間儲存或比較時間戳記，在跨時區或伺服器時區變更時產生資料錯誤

### Major
- 字串比較未指定 locale / collation，在不同語系環境行為不一致
- 硬寫環境相關設定（URL、帳號、flag）而非透過環境變數或配置注入，導致設定無法在不同環境間替換

### Minor
- 惰性求值的集合在同一資料集上多次 iterate，應先具體化（materialize）固定結果以避免重複計算或副作用

---

## 十二、邊界條件與防禦性處理（Edge Cases & Defensive Handling）

### Critical
- 對集合直接存取第一個或特定索引元素而未先確認有元素，空集合時會拋出 exception 且無錯誤上下文
- 除法運算未檢查除數為零

### Major
- 字串輸入未處理空字串（`""`）與純空白（`"   "`），僅檢查 null
- 數值輸入未處理負數或零的邊界（如計算百分比、金額、數量）
- 迴圈邊界：off-by-one 錯誤（`<` vs `<=`、index 起始點）
- 外部輸入的列舉值或整數未驗證是否在合法範圍內，直接用於分派邏輯而無 default 處理

---

## 十三、系統思維（Systems Thinking）

### Critical
- 修改 shared service 的方法簽章、DB schema 或 API response 結構，未評估所有呼叫方的影響，導致其他模組靜默錯誤或資料損毀
- 改動範圍與實際影響範圍嚴重不對稱（diff 只改 3 行，卻影響整個 domain 的核心行為），且 commit message 或 PR 說明完全未提及潛在影響
- **業務意圖不符**：PR 的程式碼實作與 PR 描述或關聯 Ticket 的需求規格明顯不符，或 Reviewer 無法從 PR 資訊推斷實作意圖；此情況應退回要求補充說明，而非帶疑問 approve

### Major
- Local optimization 造成 global regression：例如新增 index 加速特定查詢，但顯著拖慢整體 write throughput，未在 PR 中說明 tradeoff
- Cross-cutting concerns（auth、logging、cache、error handling）在不同模組各自重複實作，行為不一致，應統一至共用層
- 設計只考慮當前資料量，缺乏 scale 意識：現在 100 筆可運作，但邏輯在 10 萬筆時需要完全重寫（如全表載入後記憶體過濾）
- 跨服務邊界的資料格式假設無任何保護（無 contract test、無 schema validation），依賴方格式變動時會靜默爛掉

### Minor
- 改動涉及多個 bounded context 但未說明設計決策，reviewers 無法判斷這是刻意設計還是邊界劃分錯誤
- 新功能未考慮與既有 convention（命名、錯誤碼、response 結構）的一致性，形成系統內的孤島

### 未深讀現有 Code / 文件（Not Reading Before Writing）
- **Major**：重新實作 codebase 中已存在的 utility / helper / service，造成重複邏輯並行維護
- **Major**：同一檔案或模組已有明確的 pattern，新增的 code 自成一套，風格與結構不一致
- **Major**：呼叫或 override 方法前未讀其 contract（參數語意、回傳值、前置條件），導致錯誤使用
- **Minor**：文件或註解明確標示的限制條件（如「此方法非 thread-safe」、「需先呼叫 Init」），commit 中未遵守且無說明

### 技術債迴避（Avoiding Ownership）
- **Major**：新功能建立在已知有缺陷的舊邏輯上，未修正底層問題，且無說明為何不一併修正
- **Major**：用 flag parameter（如 `isNewFlow = true`）區分新舊行為，而非統一邏輯——這是 Fear of touching 的典型症狀
- **Major**：在舊方法旁邊新增幾乎相同的方法（如 `GetUserV2`），而非重構既有方法，導致兩套邏輯並行維護
- **Minor**：PR 中明確繞過已知問題（workaround）而非修正，且未開 follow-up issue 或說明原因

---

## 十四、測試（Testing）

### Major
- 涉及業務運算、狀態轉換或金額計算的核心邏輯，未附帶對應的單元測試
- 測試案例彼此依賴特定執行順序，或未隔離外部依賴（DB、第三方 API），導致測試環境不穩定、CI flaky

### Minor
- 測試命名未能描述測試意圖（如 `testMethod1`、`test1`）；建議採用 Given/When/Then 或 Should/When 格式清楚說明前提、操作與預期結果
