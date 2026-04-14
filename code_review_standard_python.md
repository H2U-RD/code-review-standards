# Code Review 標準規範（Python / Django / Wagtail）

本規範收錄 Python / Django / Wagtail 生態特有的常見錯誤，適用於以 Python 為主的後端服務 review。Django ORM、Wagtail Page model 相關陷阱標示於各小節。

---

## P1. Python 語言陷阱

| 等級 | 描述 |
|------|------|
| **Critical** | 使用可變物件作為函式預設參數（`def f(items=[])`）：預設值在 module 載入時建立一次，所有呼叫共用同一個物件，造成跨呼叫的狀態污染；應改用 `None` 並在函式內初始化 |
| **Critical** | Late-binding closure：迴圈內的 lambda / 巢狀函式捕捉迴圈變數（`for i in range(n): fns.append(lambda: i)`），所有函式在呼叫時都讀到最終值；應用預設參數固定（`lambda i=i: i`） |
| **Major** | 捕捉裸 `except:` 或 `except Exception:` 後靜默吞掉，包含 `KeyboardInterrupt`、`SystemExit` 在內的所有例外皆被掩蓋；應捕捉最小範圍的具體型別 |
| **Major** | 直接比較型別用 `type(x) == Foo` 而非 `isinstance(x, Foo)`，不考慮繼承，且無法配合 duck typing |
| **Minor** | `is` 用於比較值（`if x is "active"`），`is` 僅保證 CPython 的 small int / interned string，不應依賴；值比較應使用 `==` |
| **Minor** | 在大型迴圈中用 `+` 串接字串，應改用 `"".join(parts)` 或 `io.StringIO` |

---

## P2. 時間與時區

| 等級 | 描述 |
|------|------|
| **Critical** | 使用 `datetime.datetime.now()` 而非 `datetime.datetime.now(tz=datetime.timezone.utc)` 或 Django 的 `django.utils.timezone.now()`，在 `USE_TZ=True` 環境下儲存 naive datetime 會造成時區錯誤 |
| **Critical** | 直接比較 aware 與 naive datetime 物件，Python 會拋 `TypeError`；未在 view 或 serializer 入口統一轉換 |
| **Major** | 直接操作 `settings.TIME_ZONE` 轉換時間而非使用 `django.utils.timezone.localtime()`，在動態時區需求下行為不可預測 |

---

## P3. Django ORM

| 等級 | 描述 |
|------|------|
| **Critical** | 在迴圈內對關聯物件逐一查詢（N+1）且未使用 `select_related()`（ForeignKey / OneToOne）或 `prefetch_related()`（ManyToMany / reverse FK）合併 |
| **Critical** | 多個相關 DB 寫入未使用 `transaction.atomic()` 包覆，部分失敗造成資料不一致 |
| **Major** | QuerySet 在同一 request 內被多次 evaluate（如先轉 `list()` 又再次過濾），應在需要的時機才固定，或用 `_result_cache` 確認是否已快取 |
| **Major** | 使用 `Model.objects.all()` 後在 Python 層過濾（`[x for x in qs if x.field == val]`），應將過濾條件下推至 ORM（`.filter(field=val)`） |
| **Major** | 僅需特定欄位的查詢未使用 `.values()` / `.values_list()` / `.only()` / `.defer()`，回傳完整 model instance 造成不必要的資料傳輸與 object hydration |
| **Major** | 大量批次寫入用逐筆 `Model.save()`，應改用 `bulk_create()` / `bulk_update()` 減少 round trip |
| **Major** | `get_or_create()` / `update_or_create()` 未在 `transaction.atomic()` 內執行，在高並發情境有 race condition 風險 |
| **Minor** | 唯讀查詢未考慮使用 `.iterator()` 處理大結果集，避免一次將全部 row 載入記憶體 |
| **Minor** | 查詢缺少筆數上限保護（未加 `[:n]` 或分頁），production 資料量增長後可能回傳百萬筆 |

---

## P4. Django View 與序列化

| 等級 | 描述 |
|------|------|
| **Critical** | 使用者輸入直接用於 `raw()` 或 `extra()` SQL 片段，有 SQL Injection 風險；應使用 ORM 或參數化查詢 |
| **Critical** | 敏感欄位（密碼、token）未從 serializer 的 `fields` 排除或標注 `write_only=True`，可能透過 API 回傳給用戶端 |
| **Major** | 直接回傳 Django model instance 序列化結果（`model.__dict__`、`values()`）作為 API response，domain 與 API contract 強耦合；應使用專用 serializer / DTO |
| **Major** | Form / Serializer 的 `validated_data` 未經使用直接存取 `request.data` / `request.POST`，跳過驗證層 |
| **Minor** | View 內含業務邏輯（複雜計算、多步驟 DB 操作），應移至 service layer 以利測試與重用 |

---

## P5. Django 設定與安全

| 等級 | 描述 |
|------|------|
| **Critical** | `SECRET_KEY`、資料庫密碼、第三方 API key 硬寫在 `settings.py` 或 commit 進版控；應從環境變數或 secret manager 讀取 |
| **Critical** | `DEBUG=True` 可能被帶進 production，洩露 stack trace 與 settings 細節給用戶端 |
| **Major** | `ALLOWED_HOSTS` 設為 `["*"]`，未限制合法 host，有 HTTP Host header injection 風險 |
| **Major** | CSRF protection 被 `@csrf_exempt` 關掉，且未補充等效驗證（如 token auth）；state-changing endpoint 應保持 CSRF 保護 |
| **Major** | 使用 `python-decouple` / `django-environ` 之類的套件讀取環境變數，但未在 CI/CD 或部署文件中記錄必填變數，導致部署時 silent failure |
| **Minor** | `INSTALLED_APPS` 包含 `django.contrib.admin` 且 admin URL 為預設 `/admin/`，建議更換路徑或加上額外驗證層降低暴力攻擊面 |

---

## P6. Wagtail Page Model

| 等級 | 描述 |
|------|------|
| **Critical** | 覆寫 `Page.save()` 但未呼叫 `super().save(*args, **kwargs)`，Wagtail 內部的版本控制、revision、slug 更新等邏輯會被跳過 |
| **Critical** | 直接修改 `Page` instance 並呼叫 `save()` 而非走 `page.save_revision().publish()`，變更不會建立 revision 紀錄，無法回滾 |
| **Major** | 在 `Page.get_context()` 內執行大量 DB 查詢未加 `select_related` / `prefetch_related`，每次頁面渲染都產生 N+1 |
| **Major** | `content_panels` 列出的欄位與 model fields 不同步（欄位已移除但 panel 未更新，或反之），造成 Wagtail admin 顯示錯誤或資料遺失 |
| **Major** | `StreamField` block 類型在 migration 前已變更結構，舊資料的 JSON 與新 block schema 不相容，未寫資料遷移 |
| **Major** | Page model 的業務邏輯直接寫在 `serve()` 或 template tag 中，應移至 model method 或 service layer |
| **Minor** | 未設定 `parent_page_types` 與 `subpage_types`，允許任意頁面類型互相巢狀，容易在 editor 操作失誤時破壞內容樹結構 |
| **Minor** | Wagtail hook（`@hooks.register`）副作用複雜但無測試，hook 的觸發時機與 Django signal 不同，容易造成隱性耦合 |

---

## P7. Wagtail Image 與 Media

| 等級 | 描述 |
|------|------|
| **Major** | 在 template 中直接輸出 `image.file.url` 而非使用 `{% image img width-800 %}` tag，繞過 Wagtail 的 rendition 快取與格式轉換，造成效能問題且無法統一管理尺寸 |
| **Major** | 自訂 `AbstractImage` / `AbstractRendition` 後未更新 `WAGTAILIMAGES_IMAGE_MODEL` 設定，Wagtail 仍使用內建 model，自訂欄位無法使用 |
| **Minor** | 大量 rendition 在 template 迴圈內逐一生成未預先 prefetch，應使用 `{% image_url %}` 搭配 `prefetch_renditions` 或批次預生成 |

---

## P8. 非同步與背景任務

| 等級 | 描述 |
|------|------|
| **Critical** | Celery task 直接傳入 Django model instance 作為參數——序列化時會快照當下狀態，task 執行時資料可能已過時；應只傳 primary key，在 task 內重新查詢 |
| **Critical** | Celery task 未設定 `max_retries` 與 retry backoff，在外部服務不可用時會無限重試，耗盡 worker 資源 |
| **Major** | 長時間執行的 Celery task 未處理 `SoftTimeLimitExceeded` exception，worker 被強制終止時無法做清理 |
| **Major** | Django async view（`async def`）內呼叫同步 ORM（`Model.objects.filter()`）未使用 `sync_to_async()`，在 ASGI 環境下會阻塞 event loop |
| **Minor** | Celery beat 排程任務缺少冪等性保護，重複執行時可能產生重複資料或副作用 |

---

## P9. 測試

| 等級 | 描述 |
|------|------|
| **Major** | 使用 `mock.patch` 替換 ORM 查詢而非使用測試資料庫，mocked test 通過但 production migration 失敗（與過往事故一致） |
| **Major** | Wagtail Page 測試未使用 `wagtail.test.utils.WagtailPageTests` 提供的 helper，手動組裝 page tree 容易漏掉必要的 parent page 設定 |
| **Minor** | 測試使用 `from django.test import TestCase` 但場景涉及 Celery task，應改用 `CELERY_TASK_ALWAYS_EAGER=True` 或 `task_always_eager` fixture 確保 task 同步執行 |

---

## P10. 套件安全

| 等級 | 描述 |
|------|------|
| **Info** | 建議定期執行 `pip audit` 或 `safety check` 掃描已知 CVE |
| **Minor** | 新增 PyPI 套件未審查 license、維護狀態（最後 release 日期、open issues）與已知 CVE |
| **Minor** | `requirements.txt` 或 `pyproject.toml` 未鎖定版本（使用 `>=` 而非固定版本），CI 與 production 安裝結果可能不同；建議搭配 `pip-compile` 或 Poetry lock file |
