# Code Review 標準規範（Mobile：Android / iOS / React Native）

## 嚴重等級定義

| 等級 | 說明 | Merge 條件 |
|------|------|-----------|
| **Critical** | 會造成 crash、資安漏洞、資料遺失、嚴重效能問題 | 必須修正，不得 merge |
| **Major** | 顯著影響可維護性、效能、電池或使用者體驗 | 原則上必須修正，或提出書面理由後由 reviewer 裁量放行 |
| **Minor** | 建議改善，不阻擋功能 | 可於後續 PR 修正，需開 follow-up issue |
| **Info** | 風格建議、學習資源、觀察備註 | 不要求回應 |

---

## 通用 Mobile 標準

### 一、執行緒安全（Main Thread）

#### Critical
- UI 更新在背景執行緒執行（非 Main thread），會造成 UI 錯誤或 crash
- 網路請求、DB 查詢、檔案 I/O 在 Main thread 執行，阻塞 UI，造成 ANR（Android）或 frozen UI（iOS）

#### Major
- 背景任務結果回到 Main thread 的時機不明確，可能在元件銷毀後才回呼，造成 crash 或 memory leak
- 共享可變狀態在多執行緒存取而未加任何同步保護

---

### 二、Memory Management

#### Critical
- **Memory leak**：持有 Activity / Fragment / ViewController 的 static 或長生命週期參照，元件已銷毀但記憶體未釋放
- Listener、callback、observer 在元件銷毀後未取消訂閱，持續收到事件並可能操作已釋放的 UI

#### Major
- 大型 Bitmap / Image 未做 resize 或 sampling，直接載入原始尺寸造成 OOM
- 使用完畢的資源（stream、cursor、DB connection）未在 finally / defer 確保釋放

---

### 三、安全性

#### Critical
- 敏感資料（token、密碼、個資）以明文寫入 log，可在 logcat / Console 直接讀取
- 敏感資料儲存於不安全位置（Android `SharedPreferences` 明文、iOS `UserDefaults`），應使用 Keystore / Keychain
- Deep link / URL scheme 參數未驗證直接使用，有 URL injection 或 open redirect 風險
- WebView 載入外部 URL 時，若對不受信任內容暴露 `JavascriptInterface`（Android）、啟用不必要的 JavaScript 執行／原生 bridge（如 message handler）或未限制可導覽的 URL 範圍，可能導致 XSS、任意腳本呼叫原生能力或釣魚頁面風險

### Major
- 網路請求未啟用 certificate pinning，有中間人攻擊風險（視 app 敏感程度調整）
- API response 中的敏感欄位被寫入本地 cache 或 DB 而未加密
- 截圖保護未啟用（金融、醫療類 app 應設定 `FLAG_SECURE` / `.allowsScreenRecording`）

### Minor
- 除錯用的 `BuildConfig.DEBUG` flag 邏輯遺留在 release build 路徑中

---

## 四、網路與離線處理

### Major
- 網路請求未設定 timeout，依賴方無回應時無限等待
- 無任何離線狀態處理，網路斷線時直接 crash 或白畫面，沒有 fallback UI
- API 錯誤（4xx、5xx、timeout）未分類處理，全部顯示相同錯誤訊息或靜默失敗
- 重試邏輯在迴圈內無退避策略（exponential backoff），短時間內大量重打造成伺服器壓力

### Minor
- 重複的相同請求未做 deduplication 或 cache，每次進入畫面都重新 fetch

---

## 五、App Lifecycle 處理

### Major
- 未處理 app 進入背景 / 前景的狀態恢復，造成資料遺失或 UI 狀態錯誤
- 系統低記憶體或 process 被殺後重啟，未能正確還原使用者操作狀態
- 旋轉螢幕或分割畫面造成 configuration change，重新建立元件後資料遺失

### Minor
- 背景執行持續佔用資源（位置、感測器、timer）而未在適當時機暫停，造成電池耗損

---

## 六、效能與使用者體驗

### Major
- 列表元件（RecyclerView、UITableView、FlatList）未做 view recycling 或 cell reuse，大量資料時 scroll 卡頓
- 圖片未做 lazy loading 或 cache，每次滑動都重新下載
- 動畫或過場在低階裝置掉幀，未測試不同效能等級的裝置

### Minor
- 初次啟動時一次載入所有資料，應改為分頁或 lazy load
- 未針對深色模式（Dark Mode）測試 UI 顯示正確性

---

## 七、程式碼品質與可維護性

### Major
- 業務邏輯寫在 View 層（Activity / Fragment / View / Component），應移至 ViewModel / Presenter / Service
- Magic number / string 散落在 UI code（hardcoded color hex、固定 pixel 值、hardcoded API path）
- 平台專屬 code 與共用邏輯混寫，難以跨平台維護或測試

### Minor
- 過深的 callback nesting（callback hell），應改用 coroutine / async-await / Combine
- `TODO` / `FIXME` 留在 code 超過一個 sprint 未處理且無對應 issue

---

## Appendix A：Android / Kotlin

### A1. Coroutines 與非同步

### Critical
- 使用 `GlobalScope.launch`：生命週期不受控，元件銷毀後仍繼續執行，造成 memory leak 與 crash；應使用 `viewModelScope` 或 `lifecycleScope`

### Major
- 在 `viewModelScope` 以外的 scope 啟動 coroutine 而未綁定 lifecycle，無法自動取消
- `suspend` function 內使用 blocking call（`Thread.sleep`、`runBlocking`），阻塞 coroutine thread pool
- Exception 在 coroutine 內未被 `try/catch` 或 `CoroutineExceptionHandler` 處理，靜默失敗

### Minor
- `StateFlow` / `SharedFlow` collect 時未在 `Lifecycle.State.STARTED` 以上，app 在背景仍持續收到更新

## A2. MVVM 架構

### Critical
- ViewModel 持有 `Context`（尤其是 `Activity` context），造成 memory leak；需要 context 應使用 `ApplicationContext` 或 `AndroidViewModel`

### Major
- LiveData / StateFlow 在 View 層直接被修改（應只透過 ViewModel 的方法修改）
- ViewModel 直接持有 View 的參照或呼叫 View 的方法，破壞單向資料流
- Repository 層缺失，ViewModel 直接呼叫網路或 DB，無法單獨測試

### Minor
- `MutableLiveData` / `MutableStateFlow` 公開暴露（應對外只暴露 immutable 版本）

## A3. Kotlin 特有

### Critical
- `!!` 強制 non-null 用於來自外部輸入或 API 的資料，null 時直接 crash；應使用 `?.` 或 `?: `

### Major
- `lateinit var` 在未初始化時被存取，拋出 `UninitializedPropertyAccessException`；應使用 `lazy` 或 nullable
- `data class` 的 `equals`/`hashCode` 依賴可變欄位，用於 set / map key 時行為不可預測

### Minor
- 可用 `data class` 表達的 model 用一般 `class`，缺少自動生成的 `equals`/`copy`/`toString`

## A4. Android 特有

### Major
- `RecyclerView.Adapter` 未使用 `DiffUtil`，資料更新時全部 `notifyDataSetChanged()`，造成閃爍與效能問題
- 敏感資料存入 `SharedPreferences` 明文，應使用 `EncryptedSharedPreferences`
- Background task 使用 `AsyncTask`（已廢棄）或裸 `Thread`，應改用 `WorkManager` 或 coroutine

### Minor
- ProGuard / R8 rules 未覆蓋新增的第三方 library，release build 可能因 minification 造成 crash

---

## Appendix B：iOS / Swift / SwiftUI

### B1. Memory Management

#### Critical
- **Retain cycle**：closure 內強參照 `self` 而 `self` 又持有該 closure，造成兩者永遠不釋放；`@escaping` closure 應使用 `[weak self]` 或 `[unowned self]`
- `unowned` 用於生命週期不確定的參照，被參照物件釋放後存取直接 crash；不確定時應用 `weak`

#### Major
- `@StateObject` 用於從父層傳入的物件（應用 `@ObservedObject`），造成每次父層 re-render 都重新初始化

### B2. SwiftUI 架構

#### Critical
- 業務邏輯、網路請求直接寫在 `View` 的 `body` 內，應移至 `ViewModel`（`ObservableObject`）

### Major
- `@State` 儲存複雜業務狀態（應只用於 local UI state，如 `isLoading`、`isExpanded`）
- `@EnvironmentObject` 在未注入的 View 中使用，runtime 直接 crash 且無編譯期警告
- `View body` 過於複雜（超過 100 行），應拆分為子 View 或 `ViewBuilder` function
- `@Published` 屬性的更新未在 Main thread 執行，造成 UI 更新警告或 crash；應使用 `@MainActor` 或 `DispatchQueue.main`

### Minor
- `ForEach` 缺少穩定的 `id`（使用 index 或非唯一值），資料變動時動畫與狀態錯誤

## B3. Swift 特有

### Critical
- Force unwrap `!` 用於 API response 或使用者輸入的可選值，nil 時直接 crash

### Major
- `async/await` 函式在 `Task {}` 內未處理 cancellation，畫面已關閉後仍繼續執行
- `Actor` 的 isolated state 在非 actor context 直接存取，破壞 actor 隔離保證

### Minor
- `guard let` / `if let` 可以合併的多個 optional binding 分開寫，增加縮排層數

## B4. iOS 特有

### Major
- 敏感資料存入 `UserDefaults`，應使用 Keychain（`Security` framework 或 `KeychainAccess`）
- `URLSession` 直接使用而非透過統一的 network layer，無法統一加 auth header、retry、logging
- Push notification 的 payload 包含敏感個資（應只放 notification ID，資料從 API 拉取）

### Minor
- `Info.plist` 的權限說明字串（`NSCameraUsageDescription` 等）語意不清，App Store 審核可能被拒

---

# Appendix C：React Native

## C1. 效能

### Critical
- JS thread 執行 heavy computation（圖片處理、加解密、大量資料運算），阻塞所有 UI 互動；應移至 native module／JSI、`react-native-reanimated` worklet、原生背景執行緒或後端服務。若需 worker 類機制，需額外方案支援，並非 React Native 預設可直接使用瀏覽器 `Web Worker` API

### Major
- `FlatList` / `SectionList` 缺少 `keyExtractor`、`getItemLayout` 或 `windowSize` 設定，長列表 scroll 效能差
- 頻繁的 JS-to-Native bridge 呼叫（在動畫迴圈或 scroll handler 內），應使用 `Animated` API 或 `react-native-reanimated` 在 native thread 執行
- 元件在 parent re-render 時不必要地重新 render，應使用 `React.memo`、`useMemo`、`useCallback`

### Minor
- 圖片使用 `require()` 載入大尺寸本地圖片未指定 `width`/`height`，造成 layout shift

## C2. WebView 安全性

### Critical
- WebView 載入外部或使用者提供的 URL 而未做白名單驗證，有跳轉至惡意網頁的風險
- `onMessage` 接收 WebView postMessage 未驗證來源（`nativeEvent.url`），直接執行收到的指令
- WebView 注入的 `injectedJavaScript` 內容包含使用者輸入，有 XSS 風險

### Major
- WebView 與 React Native 之間傳遞敏感資料（token、個資）透過 URL query string，會被記錄在 log 中
- `javaScriptEnabled={false}` 的 WebView 被改為 `true` 而未說明原因及安全評估

## C3. 架構與平台差異

### Major
- 平台專屬邏輯（`Platform.OS === 'ios'`）散落各處而未封裝，同一個判斷在多個元件重複
- Native module 在 JS 端呼叫後未處理 Promise rejection，錯誤靜默失敗
- 直接操作 `NativeModules` 而非透過封裝的 hook / service，難以測試與替換

### Minor
- `StyleSheet.create` 定義的樣式與 inline style 混用無一致規範
- Hermes engine 不支援的 JS 語法未在 CI 驗證，可能在 Android release build 才發現問題

## C4. 工程紀律

### Major
- `package.json` 的 native dependency 版本更新後未執行 `pod install`（iOS）或重新 link（Android），造成 native / JS 版本不一致
- 新增 native module 後未更新 `.gitignore` 或建置說明，其他開發者無法 build

### Minor
- Metro bundler cache 造成的問題以「重新執行就好」帶過，未找出根本原因
