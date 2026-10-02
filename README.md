# twse-data-new — 台股資料源

本 repo 只存放資料，供儀表板、排程與 Claude 讀取。程式、觀察清單與系統架構在私有 repo。

最後更新：2026/10/02

---

## 檔案總覽

| 檔案 | 內容 | 範圍 | 更新（台北，平日） |
|---|---|---|---|
| `latest_prices.json` | 最新報價與到價提醒 | 觀察清單 | 08:00–14:45 每 15 分鐘 |
| `fundamentals.json` | 籌碼面與基本面（8 項） | 觀察清單 | 17:00 |
| `kline_history.json` | 日 K 線，保留 120 個交易日 | 觀察清單 | 17:30、21:13、隔日 07:43 |
| `material_news.json` | 重大訊息與 AI 判讀 | 觀察清單 | 盤中每 30 分鐘、16:00、20:00 |
| `radar-baseline.json` | 潛力雷達名單與評分基準 | 人工名單 | 人工＋每月／每季排程 |
| `market/` | 全市場每日快照、產業別、個股特徵 | 上市櫃全部普通股 | 17:13、21:23、隔日 07:53 |
| `themes/` | 題材雷達結果、報告、題材卡 | 全市場 | 同上 |

觀察清單以私有 repo 的 `prices.json` 為準，**加減股票只要改 `prices.json`**，各檔下次排程自動更新。

```
私有 tw-stock-alert（排程與程式）
  ├─ 觀察清單 prices.json ─┬─ check-prices        → latest_prices.json ＋ 到價 Discord
  │                       ├─ check-fundamentals  → fundamentals.json
  │                       ├─ check-kline         → kline_history.json
  │                       └─ check-material      → material_news.json ＋ 重訊 Discord
  ├─ 全市場 ─────────────── check-market        → market/ → themes/ ＋ 題材 Discord（選用）
  └─ 人工維護 ───────────── radar-baseline.json、themes/config.json、themes/kb/themes.json
                                   │
                                   ▼  本 repo（twse-data-new）
         ┌─────────────────────────┼─────────────────────────────┐
         ▼                         ▼                             ▼
  儀表板 /（到價＋清單）       /radar.html（潛力雷達）           Claude skills
  latest_prices.json         radar-baseline.json            realtime-twse-price：latest_prices
                             ＋ latest_prices.json          taiwan-stock-advisor：fundamentals、kline_history
                                                            theme-radar：themes/、market/
```

**讀取方式**：`https://raw.githubusercontent.com/samanthaYYY/twse-data-new/main/<檔案路徑>`（Claude 請用 bash curl，不要用 web_fetch）。raw 失敗時可改用 `https://cdn.jsdelivr.net/gh/samanthaYYY/twse-data-new@main/<檔案路徑>`，但 jsDelivr 有快取延遲。

**日期**：所有資料的日期都是**資料本身的交易日**，不是排程執行日；排程一天會補跑多次，重跑不會產生重複資料。

---

## latest_prices.json

以股票代號為 key：

```json
{ "2356": { "name": "INVENTEC CORP", "price": 59.7, "prevClose": 59.5, "change": 0.2, "changePercent": 0.34,
            "marketDate": "2026-10-01",
            "alertTargets": [{ "direction": "above", "target": 70 }], "updatedAt": "2026-10-01T06:46:53Z" } }
```

- `marketDate`：這筆價格對應的交易日（台北）。休市日 `updatedAt` 會更新，但 `marketDate` 仍是最後交易日。
- `alertTargets`：`above` 通常是賣出目標、`below` 通常是買入目標；空陣列代表只觀察。
- 抓不到報價的股票不會出現在檔案中。

## fundamentals.json

`data[代號]` 底下 8 個欄位，**保留官方原始物件**（欄位名稱為官方中文或英文），每筆有 `_market`（TWSE／TPEx）。頂層 `sources` 記錄各項的資料日期、`isStale`（當天無資料、改用前一交易日）與錯誤。

| 欄位 | 內容 | 頻率 |
|---|---|---|
| `institutionalFlow` | 三大法人買賣超 | 每日 |
| `marginTrading` | 融資融券餘額 | 每日 |
| `foreignHolding` | 外資持股比率 | 每日 |
| `valuation` | 本益比、殖利率、股價淨值比 | 每日 |
| `monthlyRevenue` | 月營收與年增、月增率 | 每月 |
| `dividend` | 股利分派 | 每年 |
| `insiderHolding` | 董監持股與設質 | 每月 |
| `shareholdingDistribution` | 集保股權分散 | 每週 |

**集保股權分散**：`{資料日期, levels:[...]}`，`levels` 為持股分級 1–17，每級含人數、股數、占集保比例。**分級 15＝1,000 張以上（千張大戶）**，17＝合計。週資料，平日內容不變屬正常。

## kline_history.json

```json
{ "meta": { "2356": "TWSE", "5483": "TPEx" },
  "tickers": { "2356": { "2026-10-01": { "open": 59.6, "high": 59.8, "low": 59, "close": 59.7, "volume": 8414000 } } },
  "dataQuality": { "twseRepairedAt": "...", "twseRepairedMonths": [...], "tpexRepairedAt": "...", "tpexRepairedMonths": [...] },
  "updatedAt": "..." }
```

- `volume` 單位為**股**。
- 上市股約在收盤當晚、上櫃股約在收盤後 1 小時內寫入；官方上市端點常延遲到隔天早上，所以最新一天的上市 K 棒可能要到隔日 07:43 才出現。
- `dataQuality` 記錄以官方月資料修復過的月份。2026/10/01 已修復 6 月以來上市櫃全部 K 線（舊版曾把資料整段標錯一天）。
- `meta` 可能留有已移出觀察清單的代號，讀取時以 `latest_prices.json` 的代號為準。

## material_news.json

```json
{ "updatedAt": "...",
  "seen":   ["代號|發言日期|發言時間|主旨前30字", "..."],
  "recent": [{ "code": "2356", "name": "英業達", "date": "...", "time": "...", "subject": "...",
               "aiTag": "利多", "aiNote": "一句話重點", "pushedAt": "..." }] }
```

- `seen` 為去重紀錄（保留約 1,200 筆），`recent` 為近期明細（保留 300 則）。
- `aiTag`（利多／利空／中性）與 `aiNote` 為 AI 研判，僅供參考。**2026/10 以前的紀錄為 `null`**（當時尚未啟用 AI 判讀）。

## radar-baseline.json（潛力雷達）

個股層級的潛力評分名單，供 `/radar.html` 儀表板使用；與 `themes/` 的題材雷達是不同的東西。

- 每檔欄位：`ts` 題材（1–3）、`lb` 低基期（1–3）、`dr` 需求兌現（0–2）、`ru` 已漲（0–2）、`bt` 進場策略（near／mid／deep）、`risk`（1–3）、`p` 進場錨定基準價。
- **潛力分**＝題材×1.1＋低基期×1.1＋需求兌現×0.8＋(2−已漲)×0.9。
- **進場區間**（以 `p` 為錨）：near ×0.93–1.00、mid ×0.86–0.94、deep ×0.80–0.88；停損＝區間下緣×0.93。屬指引，非精準價位。
- 由兩個 Cowork 排程維護：每月 11 日重錨 `p` 並掃月營收；4／5／8／11 月 20 日依財報重評。名單只增不自動刪除，除權息季基準價重錨屬正常。

---

## market/ — 全市場資料

| 檔案 | 內容 |
|---|---|
| `market/index.json` | `dates` 已收錄交易日、`closed` 休市日、`partial` 法人資料待補日期 |
| `market/daily/YYYY/YYYY-MM-DD.json` | 該日全市場快照（約 1,950 檔），寫入後不再修改 |
| `market/companies.json` | 代號 → 名稱、產業別、最新月營收年增率 |
| `market/features.json` | 全市場個股特徵：報酬率、量比、ATR、60 日區間位置、法人連買賣、訊號分數 |

**讀取順序**：先讀 `market/index.json` 取得可用日期，再讀需要的日期檔。

**每日快照**的 `rows` 為「代號 → 陣列」，欄位順序見同檔 `fields`：

| 欄位 | 說明 | 單位 |
|---|---|---|
| `mkt` | `T` 上市、`O` 上櫃 | |
| `o` `h` `l` `c` | 開高低收 | 元 |
| `chg` | 官方漲跌價差（相對參考價，已考慮除權息） | 元 |
| `vol` | 成交量 | **張** |
| `val` | 成交金額 | 百萬元 |
| `fi` `it` `dl` | 外資、投信、自營商買賣超 | 張 |

另有 `sectors`（上市類股指數 `[收盤, 漲跌幅%]`）與 `flags`（`notice` 注意股、`disposal` 處置中，只有最新交易日有值）。資料自 2026-08-20 起。

**注意單位差異**：`market/` 的量為「張」、`kline_history.json` 的量為「股」。

## themes/ — 題材雷達

| 檔案 | 內容 | 維護 |
|---|---|---|
| `themes/latest.json` | 今日異常股、族群、題材卡狀態 | 程式每日覆寫 |
| `themes/reports/YYYY/DATE.md` | 每日題材報告，可直接在 GitHub 上閱讀 | 程式每日一檔 |
| `themes/log/YYYY.jsonl` | 每日族群紀錄，供日後校準 | 程式附加 |
| `themes/config.json` | 訊號門檻與權重，只需寫要覆蓋的項目 | **人工** |
| `themes/kb/themes.json` | 題材卡：成員代號、角色、證據等級 | **人工**（Claude 可提案 `candidate`） |

- 族群有三種來源：同產業、同題材卡、跨產業共動。**跨產業共動且不屬於任何題材卡的族群會標為「未知族群候選」**，是新題材的線索。
- 雷達**只標記、不排除個股**（千元股、KY、處置中、追高區等皆為標籤），是否參與屬於選股階段的判斷。
- 欄位定義見私有 repo `docs/theme-radar-schema.md`；Claude 透過 `theme-radar` skill 解讀。

---

*本 repo 內容為研究彙整與紀錄，非投資建議。*
