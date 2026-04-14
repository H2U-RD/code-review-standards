# Code Review 標準規範（Java）

本規範收錄 Java 生態特有的常見錯誤，適用於以 Java 為主的後端服務 review。Spring / Hibernate 相關陷阱標示於各小節。

## B1. 相等性與雜湊

| 等級 | 描述 |
|------|------|
| **Critical** | 用 `==` 比較 `String` 或物件內容，應使用 `.equals()`；字串字面值比較應呼叫方在已知非 null 的一側以避免 NPE（`"constant".equals(var)`） |
| **Major** | 覆寫 `equals()` 但未同步覆寫 `hashCode()`（或反之），導致物件放入 `HashMap` / `HashSet` 後行為不一致 |
| **Minor** | 值物件（value object）未設計為 immutable（所有欄位 `final`、無 setter），外部可修改導致雜湊值變化 |

## B2. Null 安全與 Optional

| 等級 | 描述 |
|------|------|
| **Critical** | 直接呼叫 `Optional.get()` 未先確認 `isPresent()`，語意上等同直接 dereference null |
| **Major** | 方法回傳 `null` 表示「無結果」而非回傳 `Optional<T>`，迫使呼叫方在沒有型別提示下自行防禦 null |
| **Minor** | 將 `Optional` 用於欄位型別、方法參數或集合元素——`Optional` 設計僅用於回傳值，用於其他位置會造成序列化問題與設計混亂 |

## B3. 資源管理

| 等級 | 描述 |
|------|------|
| **Major** | `InputStream`、`Connection`、`PreparedStatement` 等 `AutoCloseable` 未使用 try-with-resources，例外發生時資源無法釋放 |
| **Major** | `HttpURLConnection` 或舊式 HTTP client 每次請求重新建立，未重用 connection pool（應改用 OkHttp、Apache HttpClient 或 Spring `RestTemplate` / `WebClient`） |

## B4. 時間 API

| 等級 | 描述 |
|------|------|
| **Critical** | 使用 `java.util.Date` 或 `Calendar` 進行時間計算或格式化，應改用 `java.time.*`（`LocalDate`、`ZonedDateTime`、`Instant`） |
| **Critical** | 將 `SimpleDateFormat` 宣告為 `static` 欄位並跨執行緒共用——`SimpleDateFormat` 非 thread-safe，並發使用會造成資料錯誤；應改用 `DateTimeFormatter`（immutable） |

## B5. 並發

| 等級 | 描述 |
|------|------|
| **Critical** | 在多執行緒場景使用 `HashMap`、`ArrayList` 等非 thread-safe 集合作為共享狀態，應改用 `ConcurrentHashMap`、`CopyOnWriteArrayList` 或加外部同步 |
| **Critical** | `long` / `double` 欄位在多執行緒下讀寫未加 `volatile` 或同步——JVM 不保證 64-bit 原始型別的原子寫入 |
| **Major** | 使用 `synchronized` 鎖住包含 I/O 的程式碼區塊，造成執行緒阻塞；應縮小 critical section 或改用非阻塞結構 |
| **Major** | `ThreadLocal` 使用後未在請求結束時呼叫 `remove()`，在執行緒池環境下會造成資料洩漏至下一個請求 |

## B6. 集合

| 等級 | 描述 |
|------|------|
| **Major** | 回傳內部 `List` / `Map` 的直接引用，外部可修改封裝物件的內部狀態；應回傳 `Collections.unmodifiableList()` 或防禦性複製 |
| **Minor** | 初始化 `ArrayList` / `HashMap` 時未指定初始容量，預期會放入大量元素卻讓容器反覆擴容，造成不必要的 allocation |
| **Minor** | 用 `for` 迴圈 index 遍歷 `LinkedList`（`get(i)` 是 O(n)），應改用 iterator 或 `for-each` |

## B7. 例外處理

| 等級 | 描述 |
|------|------|
| **Critical** | `catch (Exception e) {}` 完全吞掉例外且無任何 log，呼叫方與監攻系統均無從得知失敗 |
| **Major** | Checked exception 以 `catch` 吞掉後繼續執行，而非向上拋出或轉換為有語意的 unchecked exception |
| **Minor** | 直接 `throw new RuntimeException("message")` 而非定義有語意的自訂型別，呼叫方無法針對性 catch |

## B8. 數值與型別

| 等級 | 描述 |
|------|------|
| **Critical** | 使用浮點數（`float` / `double`）處理金額或需要精確十進位的計算，應改用 `BigDecimal` |
| **Major** | `int` 相乘後才轉 `long`（如 `(long)(a * b)`），若 `a * b` 已溢位則轉型無法救回；應先轉型再計算（`(long) a * b`） |
| **Minor** | 大量迴圈中頻繁對基本型別（`int`、`boolean`）自動裝箱（autoboxing）為包裝類別，造成不必要的物件 allocation；應確認集合或方法簽章是否可改用原始型別 |

## B9. Spring 特定

| 等級 | 描述 |
|------|------|
| **Critical** | `@Transactional` 標注在 `private` 方法上——Spring AOP proxy 無法攔截，transaction 實際不會啟動 |
| **Critical** | 在同一個 bean 內 self-invocation（`this.method()`）呼叫 `@Transactional` 方法——proxy 不介入，transaction 語意失效 |
| **Major** | `@Autowired` 使用 field injection（直接注入欄位）而非 constructor injection，導致無法在測試中替換依賴，且無法宣告欄位為 `final` |
| **Major** | Singleton bean 持有可變狀態（instance 欄位）且未加同步保護，多個並發請求共寫同一份狀態 |

## B10. Hibernate / JPA 特定

| 等級 | 描述 |
|------|------|
| **Critical** | 在 session / transaction 關閉後存取 lazy-loaded 關聯，拋出 `LazyInitializationException`；應在 transaction 內完成所有取用，或改用 `FetchType.EAGER`（需評估 N+1 風階） |
| **Major** | Entity 關聯預設 `FetchType.EAGER`（`@ManyToOne` 預設值），每次查詢都帶出關聯資料，造成隱性 N+1 或過量資料載入 |
| **Major** | 雙向關聯未同步維護兩端（只設 `parent.getChildren().add(child)` 而未設 `child.setParent(parent)`），導致 Hibernate 一級快取與 DB 狀態不一致 |
| **Minor** | 對 Entity 直接使用 `toString()`（IDE 自動生成版本）觸發 lazy 關聯載入，在 log 或 debug 時產生非預期查詢 |

## B11. 套件安全

| 等級 | 描述 |
|------|------|
| **Info** | 建議使用 `mvn dependency:check`（OWASP Dependency-Check plugin）或 Gradle 對應插件定期掃描已知 CVE |
| **Minor** | 新增 Maven / Gradle 依賴未審查 license、維護狀態與已知 CVE |
