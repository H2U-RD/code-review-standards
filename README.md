# 程式碼審查規範

工程團隊共用的程式碼審查準則集中管理庫。規範分為通用基礎規則集，以及各語言／平台的專項補充文件。

## 嚴重程度等級

| 等級 | 說明 | 合併條件 |
|------|------|----------|
| **Critical（嚴重）** | 程式錯誤、資安漏洞、資料遺失或系統崩潰 | 合併前必須修正 |
| **Major（重要）** | 對可維護性、效能、可觀測性或工程紀律有顯著影響 | 應予修正，或需提供審查者同意的書面理由 |
| **Minor（次要）** | 建議改善項目，不影響功能運作 | 可建立追蹤議題，於後續 PR 處理 |
| **Info（資訊）** | 風格建議、學習資源、觀察事項 | 無須回應 |

## 規範文件

| 文件 | 適用範圍 |
|------|----------|
| [code_review_standard.md](code_review_standard.md) | 通用 — 適用於所有語言與平台 |
| [code_review_standard_dotnet.md](code_review_standard_dotnet.md) | .NET / C# 專項 |
| [code_review_standard_java.md](code_review_standard_java.md) | Java 專項 |
| [code_review_standard_frontend.md](code_review_standard_frontend.md) | TypeScript / Vue / Nuxt |
| [code_review_standard_python.md](code_review_standard_python.md) | Python / Django / Wagtail |
| [code_review_standard_mobile.md](code_review_standard_mobile.md) | Android / iOS / React Native |

## 使用方式

1. 所有審查請先參照**通用規範** ([code_review_standard.md](code_review_standard.md))。
2. 依據 PR 的技術棧，疊加套用對應的**語言／平台補充規範**。
3. 留下審查意見時，請在每則留言前標註嚴重程度等級（例如 `[Critical]`、`[Major]`）。

## 貢獻方式

若要新增規則或修改現有規則，請開立 Pull Request（PR）/ Merge Request（MR）並包含：
- 將規則加入對應嚴重程度的章節
- 簡短說明此規則所預防的失效情境
