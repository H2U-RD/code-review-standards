# Code Review 標準規範（Frontend：TypeScript / Vue / Nuxt）

## 嚴重等級定義

| 等級 | 說明 | Merge 條件 |
|------|------|-----------|
| **Critical** | 會造成 bug、資安漏洞、資料遺失、系統崩潰 | 必須修正，不得 merge |
| **Major** | 顯著影響可維護性、效能或工程紀律 | 原則上必須修正，或提出書面理由後由 reviewer 裁量放行 |
| **Minor** | 建議改善，不阻擋功能 | 可於後續 PR 修正，需開 follow-up issue |
| **Info** | 風格建議、學習資源、觀察備註 | 不要求回應 |

---

## 一、TypeScript 型別安全

### Critical
- `any` 濫用：放棄型別檢查，等同動態語言，掩蓋潛在 runtime 錯誤
- Non-null assertion `!` 用於來自外部輸入或 API 回應的欄位（`data!.user.name`），等同不做 null check

### Major
- `as` 強制轉型（type assertion）無任何 runtime guard 保護，若資料不符預期會靜默出錯
- 外部輸入（API response、`localStorage`、`URL params`）型別應為 `unknown` 並強制驗證，不應直接用 `any` 跳過
- 函式缺少回傳型別標註，複雜邏輯下難以追蹤實際回傳值
- 使用 `// @ts-ignore` 或 `// @ts-expect-error` 壓制錯誤而未說明原因

### Minor
- `interface` 與 `type` 混用無一致規範（擇一並統一）
- Enum 使用數字值而非字串值，debug 時難以辨識

### Info
- 可啟用 `strict: true` 強制更嚴格的型別檢查

---

## 二、Vue 元件設計

### Critical
- **Props mutation**：子元件直接修改 `props.xxx`，應透過 `emit` 事件回傳父層處理，直接改會造成單向資料流破壞與難以追蹤的 bug

### Major
- `v-for` 缺少 `:key`，或 key 使用陣列 index（資料重排時 DOM diff 錯誤）
- Template 內進行複雜計算（filter、sort、map），每次 render 重算，應改為 `computed`
- God Component：單一 `.vue` 檔超過 **400 行**，邏輯未拆分為 composable 或子元件
- `watch` 做的事可以用 `computed` 表達，卻用 `watch` 造成不必要的副作用
- Composable 內的 side effect（`addEventListener`、`setInterval`、subscription）在 `onUnmounted` 未清除，造成 memory leak
- `v-if` 與 `v-show` 使用場景混淆：頻繁切換應用 `v-show`，永遠不成立的條件應用 `v-if`

### Minor
- 元件 props 缺少型別定義（未使用 `defineProps<{...}>()`）
- `emit` 未使用 `defineEmits` 宣告，其他人無法知道元件會發出哪些事件
- 過深的 prop drilling（超過 3 層），應考慮 `provide/inject` 或 Pinia store

### Info
- 建議將可重用邏輯抽為 composable（`useXxx`），保持元件專注於渲染

---

## 三、Nuxt 與 SSR

### Critical
- 直接存取 `window`、`document`、`navigator` 等瀏覽器 API 而未做 SSR 保護，server 端執行時會直接 crash（應使用 `onMounted` 或 `import.meta.client` 保護）
- **SSR hydration mismatch**：server 與 client render 的 HTML 不一致，導致畫面閃爍或 Vue 拋出 hydration 錯誤（常見於時間、隨機值、瀏覽器專屬資料）

### Major
- 需要 SEO 的頁面資料在 `onMounted` 內 fetch，server 端不會執行，爬蟲拿不到資料；應改用 `useFetch` 或 `useAsyncData`
- `/server/api/` 底下的 endpoint 未驗證身份（缺少 token 驗證或 session 檢查）
- `useAsyncData` 的 key 重複，導致不同頁面的快取資料互相覆蓋

### Minor
- 頁面缺少 `useSeoMeta` 或 `useHead` 設定 title / description，影響 SEO
- `useState` key 在多個元件重複使用不同型別，共享狀態型別不一致

### Info
- Nuxt DevTools 可協助檢查 hydration 問題與 payload 大小

---

## 四、安全性

### Critical
- **XSS**：`v-html`、`innerHTML` 直接帶入未消毒的使用者輸入或 API 資料
- **API key / secret 寫在前端**：即使放在 `.env` 也會被打包進 bundle，任何人皆可從 DevTools 或 source map 取得；只有 `NUXT_PUBLIC_` 前綴的才應暴露
- 敏感資料（JWT token、使用者個資）存入 `localStorage`，易被 XSS 攻擊竊取；應改用 `httpOnly` cookie

### Major
- `postMessage` 未驗證 `event.origin`，接受任意來源的訊息
- 動態路由參數或使用者輸入未做編碼直接插入 URL 或 DOM

### Minor
- 第三方 script 未加 `integrity` 屬性（SRI），CDN 被污染時無法偵測

---

## 五、效能

### Major
- **Bundle size**：整包引入大型 library（`import _ from 'lodash'`、`import * as icons from '@iconify/vue'`）而非 tree-shaking，造成首頁 JS 過大
- **Race condition**：多個非同步請求並發，舊請求的回應覆蓋新請求的結果（應使用 `AbortController` 或 composable 內的 cancel 機制）
- 大型列表未使用虛擬捲動（virtual scroll），一次 render 數千個 DOM 節點
- 圖片未設定 `width`/`height` 或使用 `<NuxtImg>`，造成 CLS（Cumulative Layout Shift）

### Minor
- `computed` 內進行 heavy 計算且依賴頻繁變動的 reactive data，應考慮 debounce 或 `watchEffect` + cache
- 未使用 `defineAsyncComponent` 延遲載入非首屏元件

### Info
- 建議使用 Nuxt DevTools 或 Lighthouse 定期檢查 bundle size 與 Core Web Vitals

---

## 六、程式碼品質與可維護性

### Major
- 相同邏輯在多個元件重複，未抽取為 composable 或 utility function
- 業務邏輯寫在 template 或元件內，應移至 composable / store / service layer
- Magic number / magic string 散落在 template 與 script（應宣告為具名常數或 enum）
- `console.log` / `console.error` 留在 production code

### Minor
- Composable 命名不以 `use` 開頭，不符合 Vue 社群慣例
- 過度拆分：元件只有 5–10 行且只使用一次，拆出來反而增加跳讀負擔
- `store` 直接在 template 呼叫 action，應封裝於 composable 內，方便測試與替換

### Info
- 建議統一使用 `<script setup>` 語法，避免混用 Options API 與 Composition API

---

## 七、工程紀律

### Critical
- Commit 包含機密資訊（API key、`.env` 檔案、token）

### Major
- Commit message 無法描述做了什麼（如 `fix`、`update`、`wip`）
- 直接 push to `main` 未走 MR 流程
- `package-lock.json` 或 `pnpm-lock.yaml` 未進版控，導致不同環境安裝結果不同

### Minor
- 引入新 npm 套件未審查 bundle size（可用 bundlephobia.com）、維護狀態與已知 CVE
- `.env.example` 未同步更新新增的環境變數

---

## 八、系統思維（前端）

### Major
- 修改共用元件（`BaseButton`、`AppModal` 等）未評估所有使用方的影響
- API 回應格式變動（欄位新增/移除/改名）未同步更新前端所有使用處，且無 contract 保護
- 狀態管理邊界模糊：本應是 local state 的資料放進全域 store，造成不必要的耦合
- 改動涉及 SSR / CSR 邊界但未說明設計決策，reviewer 無法判斷是否正確

### Minor
- 新功能未考慮與既有元件庫 convention（props 命名、事件命名、slot 結構）的一致性
