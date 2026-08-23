# twse-data-new — 台股報價 + 籌碼面 + K線 + 潛力雷達資料源

這裡存放外部讀取專用的資料。程式與觀察清單在私有 repo `tw-stock-alert`。

最後更新：2026/08/23（新增董監持股／集保股權分散、補記 K 線與上櫃來源、退役 Gist）

---

## 檔案說明

| 檔案 | 內容 | 由誰寫 | 給誰讀 |
|---|---|---|---|
| `latest_prices.json` | 最新報價快照（現價、昨收、漲跌、到價提醒、更新時間），key 為股票代號 | GitHub Actions（`check-prices.mjs`）每 15 分鐘 | 儀表板、排程、Claude |
| `fundamentals.json` | **八面向**籌碼/基本面快照（見下），key 為股票代號，值為官方回傳原始物件 | GitHub Actions（`check-fundamentals.mjs`）平日 17:00 | Claude（`taiwan-stock-advisor` skill） |
| `kline_history.json` | 個股日 K 線歷史（開高低收量），保留約 120 交易日 | GitHub Actions（`check-kline.mjs`）平日 17:30 | Claude（技術分析） |
| `radar-baseline.json` | 潛力雷達的**名單＋分析基準**（題材/基期/潛力標籤、`p`進場錨定基準價） | 人工維護 + 排程刷新 | 儀表板、排程、Claude |
| `README.md` | 本說明 | 人工 | 人 |

> 私有 repo 的 `prices.json`（觀察清單，62 檔）是「要抓哪些股票」的唯一源頭。**加減股票 = 改 `prices.json`**，三份輸出檔下次排程自動更新。

---

## 系統總覽

```
私有 tw-stock-alert：prices.json（觀察清單，手動編輯）＝三套排程共用的唯一入口
        │
        ├─ check.yml            平日每15分鐘  → check-prices.mjs      → latest_prices.json
        ├─ check-fundamentals   平日 17:00     → check-fundamentals.mjs → fundamentals.json
        └─ check-kline          平日 17:30     → check-kline.mjs       → kline_history.json
                （三支都寫進本 public repo，共用同一把 DATA_REPO_TOKEN）
                                     │
        ┌────────────────────────────┼───────────────────────────────┐
        ▼                            ▼                                ▼
  React 儀表板 /             潛力雷達 /radar.html               Claude skills
  （讀 latest_prices）       （讀 radar-baseline+latest_prices）  （bash curl 讀四份 json）
```

三套排程互相獨立（抓取邏輯、輸出檔、排程時間完全分開），只共用 `prices.json` 觀察清單與 `DATA_REPO_TOKEN`。

> 2026/08/23 起 `latest_prices.json` **不再雙寫 Secret Gist**：儀表板已改讀本 repo 的 raw URL，單一資料源。

---

## fundamentals.json — 八面向籌碼/基本面

給 Claude 分析個股用，取代對話當下即時打官方 API（會踩快取和網址解鎖限制）。

**結構**：`data` 以股票代號為 key，每檔底下八個欄位，全部保留官方回傳原始物件、標 `_market`（TWSE/TPEx）追溯來源。頂層 `sources` 記各項 `date`/`isStale`/`error`。

| 欄位 | 內容 | 上市 | 上櫃 |
|---|---|---|---|
| `institutionalFlow` | 三大法人買賣超 | T86 | tpex_3insti_daily_trading |
| `marginTrading` | 融資融券餘額 | MI_MARGN | tpex_mainboard_margin_balance |
| `foreignHolding` | 外資持股比率 | MI_QFIIS | tpex_3insti_qfii |
| `insiderHolding` ★ | 董監事／內部人持股 | t187ap11_L | mopsfin_t187ap11_O |
| `shareholdingDistribution` ★ | 集保股權分散（大戶/散戶分佈） | TDCC getOD.ashx?id=1-5（上市櫃通用） | 同左 |
| `monthlyRevenue` | 月營收 | t187ap05_L | mopsfin_t187ap05_O |
| `dividend` | 股利分派 | t187ap45_L | mopsfin_t187ap39_O |
| `valuation` | PE/殖利率/PB | BWIBBU_ALL | tpex_mainboard_peratio_analysis |

★ = 2026/08/23 新增的籌碼面。

**集保股權分散（`shareholdingDistribution`）讀法**：每檔一個物件 `{資料日期, levels:[...]}`，`levels` 為持股分級 1–17 的陣列，每級含 `持股分級`/`人數`/`股數`/`占集保比例`。**分級 15 = 1,000 張以上（千張大戶）**、16=差異數調整、17=合計。看「分級 15 占比」的週變化就是最直接的大戶籌碼流向。TDCC 為每週資料。

**逐日資料（T86/融資融券/外資持股）** 找不到當天資料時會自動往前找最近交易日，並用 `isStale` 標記。

**已知踩過的坑**：融資融券 (MI_MARGN) 官方 JSON 是巢狀 `tables`（大盤總計 + 逐股明細），逐股明細用「列數最多」判斷；openapi/tpex/tdcc 系列無查詢參數、永遠回傳全市場，程式自行過濾成觀察清單。

---

## kline_history.json — 日 K 線歷史

`meta` 記每檔市場別（TWSE/TPEx）；`tickers[代號][YYYY-MM-DD] = {open,high,low,close,volume}`。每日例行更新只需 2 次 call（TWSE 全市場 1 次 + TPEx 全市場 1 次）；上市新股首次會逐月回補 3 個月，上櫃股無回補（官方 API 限制，只能從開始執行當天累積）。保留約 120 交易日（涵蓋季線）。

---

## 潛力雷達儀表板（dashboard/public/radar.html，部署於 /radar.html）

**固定模板、樣式不變**：開啟時線上抓 `radar-baseline.json`（名單＋分析）＋ `latest_prices.json`（即時價）合併呈現，抓不到退 jsDelivr、再不行用內建備援。

- **潛力分** = 題材強度×1.1 + 低基期×1.1 + 需求兌現×0.8 + (2−已漲程度)×0.9
- **標籤**：`ts`題材1-3、`lb`低基期1-3、`dr`需求兌現0-2、`ru`已漲0-2、`risk`1-3
- **進場策略** `bt`：`near`近價 / `mid`回檔承接 / `deep`高檔候低
- **進場區間**（錨 `p`）：near ×[0.93,1.00]、mid ×[0.86,0.94]、deep ×[0.80,0.88]；停損＝下緣×0.93。屬指引、非精準線圖價位。**非投資建議。**

---

## 兩個排程（Cowork Scheduled）

| 排程 | 何時 | 做什麼 |
|---|---|---|
| `twse-radar-monthly-light` | 每月 11 號 09:00 | 用最新收盤重錨各檔 `p`、掃月營收異動、微調 dr/ru |
| `twse-radar-quarterly-deep` | 4/5/8/11 月 20 號 09:00 | 深查財報，重評 基期/題材/需求/是否已大漲、估值、屬性 |

共同規則：名單以 `radar-baseline.json` 為準全部保留；報價源新增=新標的加入；報價源這次缺的保留並標記、絕不自動刪；不產 HTML（儀表板固定模板）。

---

## 維護方式

- **加/減股票** → 改私有 repo 的 `prices.json`（唯一入口）。三份輸出檔下次排程自動涵蓋。
- **手動細修雷達分析** → 改 `radar-baseline.json` 重新提交。
- **儀表板** → 直接開 `radar.html`（自己線上抓最新資料）；只有模板改版才換新檔。
- **除權息季（7–8 月）** → 進場區間因基準價重錨而變動，屬正常。

---

*本 repo 內容為研究彙整與紀錄，非投資建議；進出請自行判斷、自負盈虧。*
