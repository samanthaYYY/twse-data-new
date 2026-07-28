# twse-data-new — 台股報價快照 + 潛力雷達資料源

這裡存放外部讀取專用的資料。程式與觀察清單在私有 repo `tw-stock-alert`。

---

## 檔案說明

| 檔案 | 內容 | 由誰寫 | 給誰讀 |
|---|---|---|---|
| `latest_prices.json` | 最新報價快照（現價、昨收、漲跌、到價提醒、更新時間），key 為股票代號 | GitHub Actions（`tw-stock-alert` 的 `check-prices.mjs`）每 15 分鐘 | 儀表板、排程、Claude |
| `radar-baseline.json` | 潛力雷達的**名單＋分析基準**：每檔題材/基期/潛力標籤、催化、`p`(進場區間錨定基準價) | 人工維護 + 排程刷新後回存 | 儀表板、排程、Claude |
| `README.md` | 本說明 | 人工 | 人 |

> 私有 repo `tw-stock-alert` 的 `prices.json` 是「要抓哪些股票」的源頭；`latest_prices.json` 是它跑出來的結果。**加減股票 = 改 `prices.json`**，`latest_prices.json` 下次排程自動更新。

---

## 系統總覽

```
私有 tw-stock-alert：prices.json（觀察清單，手動編輯）
        │  GitHub Actions 每15分鐘（cron-job.org 觸發 workflow_dispatch，內建 cron 備援）
        ▼
   check-prices.mjs 抓 Yahoo → 雙寫：Gist（舊 dashboard）＋ 本 repo / latest_prices.json
        │                                    │
        ▼                                    ▼
   潛力雷達儀表板（固定模板 HTML）      月/季排程（Cowork Scheduled）
   線上抓 radar-baseline + latest_prices   只更新 radar-baseline、出摘要
```

報價抓取邏輯（`check-prices.mjs`、Gist 雙寫、`alertTargets`）不受本 repo 的儀表板/排程影響——它們都只「讀」`latest_prices.json`。

---

## 潛力雷達儀表板（tw-watchlist-radar.html）

**固定模板、樣式不變**：一個 HTML 檔，開啟時自己線上抓 `radar-baseline.json`（名單＋分析）＋ `latest_prices.json`（即時價）合併呈現。不需重建、也不會被排程改樣式。抓不到時用內建備援資料。

- **潛力分** = 題材強度×1.1 + 低基期×1.1 + 需求兌現×0.8 + (2−已漲程度)×0.9
- **標籤**：`ts`題材強度1-3、`lb`低基期1-3（高=獲利未爆發上檔大）、`dr`需求兌現0-2、`ru`已漲0-2（0=還沒漲）、`risk`1-3
- **屬性/進場策略** `bt`：`near`近價布局 / `mid`回檔承接 / `deep`高檔候低
- **進場區間**（錨 `p`）：near ×[0.93,1.00]、mid ×[0.86,0.94]、deep ×[0.80,0.88]；停損＝下緣×0.93。屬指引、非精準線圖價位。
- 報價源沒有的新代號顯示「新標的」；`radar-baseline` 有而報價源暫無的，保留並標「報價源暫無」。**非投資建議。**

---

## 兩個排程（Cowork Scheduled）

| 排程 | 何時 | 做什麼 |
|---|---|---|
| `twse-radar-monthly-light` | 每月 11 號 09:00 | 用最新收盤重錨各檔 `p`、掃月營收異動、微調 dr/ru |
| `twse-radar-quarterly-deep` | 4/5/8/11 月 20 號 09:00 | 深查財報，重評 基期/題材/需求/是否已大漲、估值、屬性 |

兩者共同規則（不脫鉤、不誤刪的關鍵）：
1. **名單以 `radar-baseline.json` 為準**，全部保留。
2. 報價源新增的代號＝新標的，研究後**加入**。
3. 報價源這次缺的**保留並標記、絕不自動刪**。
4. **不產 HTML**（儀表板是固定模板）；產出刷新後的 `radar-baseline.json` 交付，供人工提交回本 repo 覆蓋。

---

## 維護方式

- **加/減股票** → 改私有 repo 的 `prices.json`（唯一入口）。新股第一次由排程自動研究，之後排程會給刷新版 `radar-baseline.json`，提交回本 repo 固定下來。
- **手動細修分析** → 改 `radar-baseline.json` 重新提交。
- **儀表板** → 用 `tw-watchlist-radar.html` 這個檔，直接瀏覽器開即可（自己線上抓最新資料）；只有模板本身改版時才需換新檔。
- **除權息季（7–8 月）** → 進場區間會因基準價重錨而變動，屬正常。

---

*本 repo 內容為研究彙整與紀錄，非投資建議；進出請自行判斷、自負盈虧。*
