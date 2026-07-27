# twse-data-new — 台股報價快照 + 潛力雷達資料源

存放兩份儀表板讀取用資料。

---

## 檔案說明

| 檔案 | 內容 | 由誰寫 | 給誰讀 |
|---|---|---|---|
| `latest_prices.json` | 最新報價快照（現價、昨收、漲跌、到價提醒、更新時間），key 為股票代號 | GitHub Actions（`tw-stock-alert` 的 `check-prices.mjs`）每 15 分鐘寫入 | 儀表板、潛力雷達排程、Claude |
| `radar-baseline.json` | 潛力雷達的「分析基準檔」：每檔的題材/基期/潛力標籤與催化 | 人工維護 + 季度排程刷新後回存 | 潛力雷達排程、Claude |
| `test.json` | 舊測試檔，可刪 | — | — |

> 觀察清單 `prices.json`（要追蹤哪些股票）在**私有** repo `tw-stock-alert`。它是名單的源頭；`latest_prices.json` 是它跑出來的結果。**要加減股票，改私有 repo 的 `prices.json` 即可**，公開的 `latest_prices.json` 下次排程就自動更新。

---

## 系統總覽

```
私有 tw-stock-alert
  prices.json（觀察清單，手動編輯）──┐
                                     │ GitHub Actions 每15分鐘
                                     ▼
  check-prices.mjs 抓 Yahoo → 雙寫：
     ├─ Gist（給舊 dashboard）
     └─ public twse-data-new / latest_prices.json ← 本 repo
                                     │
              ┌──────────────────────┼───────────────────────┐
              ▼                      ▼                        ▼
      潛力雷達儀表板            月度/季度排程              Claude 對話
   (tw-watchlist-radar.html)   (cowork Scheduled)      (realtime-twse-price skill)
```

報價排程本身由 **cron-job.org 每 15 分鐘打 GitHub `workflow_dispatch`** 觸發（GitHub 內建 schedule 不可靠，只留一條 `7,22,37,52 0-6 * * 1-5` 當備援）。

---

## 潛力雷達（tw-watchlist-radar.html）

依使用者風格排序的觀察清單儀表板：偏早期題材型成長，找「有題材與需求、但獲利低基期、還沒大漲」趁低布局。用瀏覽器打開，會即時抓 `latest_prices.json` 疊上分析。

**潛力分** = 題材強度×1.1 + 低基期×1.1 + 需求兌現×0.8 + (2−已漲程度)×0.9

**標籤**（存在 `radar-baseline.json`）：
- `ts` 題材強度 1–3
- `lb` 低基期 1–3（高＝獲利未爆發、上檔大）
- `dr` 需求兌現 0–2（純題材 / 浮現 / 已兌現）
- `ru` 已漲 0–2（0＝還沒大漲）
- `bt` 進場策略：`near` 近價布局 / `mid` 回檔承接 / `deep` 高檔候低
- `risk` 1–3

**進場區間**（固定式，錨最新收盤）：near ×[0.93,1.00]、mid ×[0.86,0.94]、deep ×[0.80,0.88]；停損＝區間下緣×0.93。屬指引，非精準線圖價位。**非投資建議。**

---

## 兩個排程（Cowork Scheduled）

| 排程 | 何時 | 做什麼 |
|---|---|---|
| `twse-radar-monthly-light` | 每月 11 號 09:00（月營收後） | 重設進場區間、掃月營收異動、自我修復新標的、重建儀表板 |
| `twse-radar-quarterly-deep` | 4/5/8/11 月 20 號 09:00（財報季後） | 深查財報、重評基期/潛力/屬性、重建儀表板、**產出刷新後的 `radar-baseline.json` 供回存** |

兩者共同邏輯（不脫鉤的關鍵）：
1. **名單**以 `latest_prices.json` 為準。
2. **分析**讀 `radar-baseline.json`。
3. 對齊：報價源有而基準檔無＝新標的自動研究；基準檔有而報價源無＝移除。

---

## 維護方式

- **加/減股票** → 改私有 repo 的 `prices.json`（只有這一處）。新股第一次由排程自動研究，之後季度排程會產出刷新版 `radar-baseline.json` 交付，**提交回本 repo** 固定下來。
- **手動細修分析** → 直接改 `radar-baseline.json` 重新提交。
- **除權息季（7–8 月）** → 進場區間會因跳空重設，屬正常。

---

*本 repo 內容為研究彙整與紀錄，非投資建議；進出請自行判斷、自負盈虧。*
