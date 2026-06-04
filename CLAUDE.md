# NCNU Opportunity Radar Agent

> 這份文件是給 Claude Code 看的專案規格。
> 將此檔放在專案根目錄命名為 `CLAUDE.md`，Claude Code 啟動時會自動讀取。

---

## 專案目標

建立一個跑在 Claude Code 上的 **AI Agent**（不是搜尋網站），幫暨南國際大學（NCNU）學生自動蒐集、判斷、推薦校內外的機會（實習、獎學金、比賽、講座）。

**核心定位**：會思考、會學習、會誠實面對自己不確定的個人助理。

## ⚠️ 這是一個 Agent，不是流水線

設計時請務必遵守以下原則，**這些是區別「Agent」與「自動化腳本」的關鍵**：

1. **自主規劃**：`/check-opportunities` 內，**由 AI 自己決定**要呼叫哪些 tool、用什麼順序、要不要再搜一輪、要不要換關鍵字。**不要寫成寫死的 step 1 → step 2 → step 3 流程**。
2. **情境感知**：每次執行時，AI 要先看使用者的 profile 和最近行為歷史，推測「使用者現在處於什麼狀態」，再決定推什麼。
3. **誠實面對不確定**：AI 對某則機會信心 < 70% 時，主動展示推理過程讓使用者可以反駁。
4. **可信度透明**：每則機會帶 source 標籤，多來源加分但**絕不過濾掉單一來源的機會**。
5. **隱性回饋**：以「點擊率」與「停留時間」作為主要學習信號，不打擾使用者手動評分。
6. **GitHub 為家**：所有狀態（profile, history）存於 GitHub 私人 repo，跨裝置同步。

---

## 技術架構

```
ncnu-opportunity-radar/
├── CLAUDE.md                    # 本檔（專案說明）
├── .mcp.json                    # MCP server 設定
├── server.py                    # MCP server 主程式
├── user_profile.json            # 使用者個人檔案（推到 GitHub 私人 repo）
├── cache/                       # 24 小時快取目錄
│   └── .gitkeep
├── tools/
│   ├── __init__.py
│   ├── scrape_school.py         # 爬學校網站
│   ├── search_web.py            # DuckDuckGo 搜尋
│   └── filter_ai.py             # AI 過濾與評分
├── .claude/
│   └── commands/
│       ├── init-profile.md      # 建立個人檔案
│       └── check-opportunities.md  # 主要功能
├── requirements.txt
├── .gitignore                   # 至少要排除 cache/、venv/
└── README.md
```

**技術棧**：
- Python 3.10+
- MCP SDK (`mcp` package)
- `requests` + `beautifulsoup4`（爬蟲）
- `duckduckgo-search`（免 API key 的搜尋）
- 透過 Claude API 做 AI 判斷
- GitHub（profile 儲存與跨裝置同步）

---

## 使用者檔案結構（user_profile.json）

```json
{
  "school": "NCNU",
  "department": "資工系",
  "department_keywords": ["程式", "軟體", "AI", "資料"],
  "grade": "大三",
  "identity": ["一般生"],
  "skills": ["Python", "React", "vibe coding"],
  "interests": ["實習", "黑客松", "AI"],
  "extra_keywords": [],
  "avoid": ["保險業"],
  "time_constraint": "寒暑假可全職",
  "show_reasoning_mode": "balanced",
  "weights": {
    "實習": 10,
    "比賽": 8,
    "獎學金": 6,
    "講座": 3
  },
  "history": {
    "clicked": [],
    "skipped": [],
    "applied": [],
    "stay_time": {},
    "feedback": []
  },
  "last_check": null,
  "profile_created": null,
  "profile_updated": null
}
```

**重要欄位說明**：
- `department_keywords`：由 AI 從科系名稱推論，**冷門系（如東南亞學系、諮輔系）AI 推論可能不準**，要讓使用者透過 `extra_keywords` 補充
- `extra_keywords`：使用者手動補充的關鍵字，搜尋時要跟 `department_keywords` 合併使用
- `show_reasoning_mode`：`always` / `balanced`（預設，AI 信心 < 70% 才展示）/ `never`
- `history.clicked` / `applied`：用於情境推理與權重學習
- `history.stay_time`：以 opportunity_id 為 key，停留秒數為 value，<3 秒視為負面信號

---

## GitHub 儲存設計

User profile 與 history 不存在本機，而是同步到使用者個人的 GitHub 私人 repo：

**初次設定流程**（在 `/init-profile` 結尾引導）：
```bash
# 在 ncnu-opportunity-radar 資料夾內
git init
git add user_profile.json
git commit -m "Initial profile"
# 使用者在 GitHub 建立 private repo 後：
git remote add origin git@github.com:USERNAME/ncnu-radar.git
git push -u origin main
```

**每次更新流程**（在 `/check-opportunities` 結尾自動執行）：
```bash
git add user_profile.json
git commit -m "Update history $(date +%F)"
git push
```

**設計理由**：
- 免費、跨裝置同步、隱私由使用者掌握
- 使用者可直接編輯 JSON（不需要 UI）
- 符合「自管 back-end」的架構建議

---

## 三個 Tool 規格

### Tool 1: `scrape_school`

**功能**：爬暨大固定來源頁面，回傳結構化資料。

**目標 URL**：
- 暨大快訊：`https://www.ncnu.edu.tw/p/403-1000-515-1.php?Lang=zh-tw`
- 實習專區：`https://psyguide.ncnu.edu.tw/p/403-1078-533-1.php?Lang=zh-tw`
- 求職專區：`https://psyguide.ncnu.edu.tw/p/403-1078-532-1.php?Lang=zh-tw`
- 工讀專區：`https://psyguide.ncnu.edu.tw/p/403-1078-534-1.php?Lang=zh-tw`

**回傳格式**：
```python
[
  {
    "title": "【實習資訊】台積電 CareerHack",
    "date": "2026-05-15",
    "url": "https://...",
    "source": "psyguide_internship",
    "raw_text": "...全文（前 500 字）...",
    "category_hint": "internship"  # 從來源 URL 推測類型
  },
  ...
]
```

**注意事項**：
- 每筆要含 source 標籤
- 用 BeautifulSoup 解析表格列
- 處理「無日期」的容錯（用 None，不要 raise）
- 加 User-Agent header 避免被擋
- 並行爬四個 URL（用 concurrent.futures）

---

### Tool 2: `search_web`

**功能**：根據 user_profile 動態組合關鍵字，做外部搜尋。

**動態關鍵字組合邏輯**：

```python
# 拿出 profile 內所有可用關鍵字
all_keywords = (
    profile["department_keywords"] +
    profile["extra_keywords"] +
    profile["interests"]
)

# 依據 weights 排序產生搜尋字串，每組必含當年度（時效檢查策略 1）
queries = []
current_year = 2026
for category, weight in sorted(profile["weights"].items(), reverse=True):
    if weight >= 5:  # 只搜權重 >= 5 的
        for kw in all_keywords[:3]:  # 每類取前 3 個關鍵字
            queries.append(f"{kw} {category} {current_year}")
```

**搜尋實作**：用 `duckduckgo-search` 套件（免 API key）：
```python
from duckduckgo_search import DDGS
with DDGS() as ddgs:
    results = list(ddgs.text(query, max_results=10, region="tw-tzh"))
```

**回傳格式**：同 `scrape_school`，但 `source` 為 `web_search`。

**時效檢查（四層機制）**：
1. **層 1：關鍵字含當年度**（已在上面實作）
2. **層 2：URL 或標題含舊年度** → 過濾掉（例如標題含 2023、2024）
3. **層 3：DuckDuckGo 提供的時間戳** → 過濾掉超過 3 個月前
4. **層 4：AI 第二次驗證**（在 filter_ai 階段做）

---

### Tool 3: `filter_ai`

**功能**：對抓到的所有機會做 AI 判斷與評分。這是 Agent 的「大腦」。

**輸入**：`(user_profile, raw_opportunities)`

**處理流程**（順序很重要）：

#### Step 1：情境推理（先做這個）

把 user_profile + 最近 10 筆點擊歷史餵給 Claude，請它推測：
- 使用者目前的階段（剛開始用 / 積極尋找 / 疲乏期）
- 最近偏好的傾向（更愛實習 / 更愛比賽 / 沒明顯傾向）
- 適合的推薦數量（積極期 8-10 則 / 一般 5-7 則 / 疲乏期 3 則）

這個推理結果會影響後面的評分權重。

#### Step 2：逐筆 AI 判斷

對每則機會做：
- **資格檢查**：「這則機會的資格條件，使用者符合嗎？」不符直接過濾
- **時效檢查（AI 二次驗證）**：「這則機會的截止日是否在今天之後？如果沒寫日期，內容是否提到 2026 年？」沒通過標記 warning
- **跨科系處理**：如果是冷門系（東南亞學系、諮輔系等），AI 要善用 `extra_keywords` 而不只看 `department`

#### Step 3：相關性評分（0-100）

```
基礎分 = 興趣關鍵字命中數 × 對應 category 權重
加分項：
  + 含獎金 +10
  + 含面試機會 +15
  + 含實習機會 +10
  + 該類別最近常被點 +5
  + 跨來源（sources >= 2）+5  ← 加分但不過濾單一來源
  + 符合 extra_keywords +8
扣分項：
  - 命中 avoid 清單 → 直接踢掉
  - 截止 < 3 天但分數 < 60 → -10（不浪費使用者時間）
```

#### Step 4：去重

相同標題（或標題相似度 > 80%）合併，`sources` 用 list 保留所有來源。

#### Step 5：信心度標記

每筆機會 AI 要自評信心度（0-100）。`< 70` 標記 `show_reasoning = True`。

**回傳格式**：
```python
[
  {
    "title": "...",
    "score": 87,
    "confidence": 85,
    "verified": True,
    "sources": ["psyguide_internship", "web_search"],
    "deadline": "2026-06-15",
    "days_left": 11,
    "ai_reason": "符合資工大三背景，含實習面試機會，興趣命中：黑客松",
    "url": "...",
    "show_reasoning": False,
    "reasoning_detail": "（信心 < 70 時填入詳細推理）",
    "category": "internship"
  },
  ...
]
```

---

## 兩個 Slash-Command

### `/init-profile`

第一次使用時建立個人檔案。詳細對話範本見 `.claude/commands/init-profile.md`（已寫好，不要改）。

收集 10 個問題後：
1. 寫入 `user_profile.json`
2. **指引使用者把 profile 推到 GitHub 私人 repo**（重要）

### `/check-opportunities`

主要功能。**注意：請不要寫成寫死的 step 1 → step 2 → step 3，要讓 AI 自主規劃**。

**設計方式**：
- 提供 AI 一個任務目標（「幫使用者找出現在最該關注的機會」）
- 列出可用的 tools 與它們的能力
- 讓 AI 自己決定先做什麼、後做什麼、要不要再做一輪
- AI 的決策過程要可被觀察（透過 log 或輸出顯示）

**典型流程**（但 AI 可以調整）：

```
1. 讀取 user_profile.json
2. 檢查 cache/（24 小時內有 cache 直接用，選做）
3. AI 規劃：依使用者狀態決定要不要爬全部來源、要不要多搜一輪
4. 並行呼叫 scrape_school 與 search_web
5. 把結果交給 filter_ai 過濾評分
6. AI 看評分結果，決定要不要為某些不確定的機會做第二次搜尋
7. 用 Markdown 格式輸出
8. 把 history 寫回 user_profile.json 並 git push
```

**輸出格式**：
```
🔥 高優先（截止 < 7 天且 score > 80）
1. 標題    [學校 ✓] [搜尋 ✓] 截止 6/10 ⭐87
   AI 理由：符合資工大三，含實習面試機會
   [展開推理] [標記已申請] [不感興趣]

📅 本月內
2. ...

📨 信箱相關（如啟用 Gmail，選做）

底部互動指令：
「展開 #N」「標記 #N 已申請」「不感興趣 #N」
```

當 `show_reasoning = True` 時，預設展開推理段落，讓使用者看到 AI 為什麼這樣判斷。

---

## 開發優先順序

### Day 1 上午（必做）
- [ ] 建立資料夾結構
- [ ] 寫 `.mcp.json` 與 `server.py` 框架
- [ ] 設計 `user_profile.json` 與寫 `/init-profile`
- [ ] 寫 `.gitignore`（排除 cache、venv、__pycache__）

### Day 1 下午（必做）
- [ ] 實作 `scrape_school`（含三個 URL，並行爬）
- [ ] 實作 `search_web`（含動態關鍵字、四層時效檢查的層 1-3）

### Day 2 上午（必做）
- [ ] 實作 `filter_ai`（情境推理 + 資格 + 時效 + 評分 + 去重 + 信心度）
- [ ] 整合 `/check-opportunities`（**用 Agent 模式，不要寫死流程**）

### Day 2 下午（必做）
- [ ] 端對端測試
- [ ] GitHub 同步機制（commit + push）
- [ ] 寫 README

### 選做（時間允許）
- [ ] 24 小時 cache 機制
- [ ] AI 信心度展示與互動展開
- [ ] 停留時間追蹤
- [ ] Gmail IMAP 整合（不用 OAuth）

---

## 第一次啟動指令

把這份 CLAUDE.md 放好之後，在 Claude Code 中說：

> 「請依照 CLAUDE.md 開始建立這個專案。從建立資料夾結構與 requirements.txt 開始，然後寫 server.py 的基礎框架。先做 Day 1 上午的部分。」

Claude Code 會自動讀取本檔並開始工作。

---

## 測試使用者（給 Claude Code 開發時用）

```json
{
  "school": "NCNU",
  "department": "資工系",
  "department_keywords": ["程式", "軟體", "AI"],
  "grade": "大三",
  "identity": ["一般生"],
  "skills": ["Python", "React", "vibe coding"],
  "interests": ["實習", "黑客松", "AI"],
  "extra_keywords": [],
  "avoid": ["保險業"],
  "time_constraint": "寒暑假可全職",
  "show_reasoning_mode": "balanced",
  "weights": {"實習":10,"比賽":8,"獎學金":6,"講座":3},
  "history": {"clicked":[],"skipped":[],"applied":[],"stay_time":{},"feedback":[]},
  "last_check": null,
  "profile_created": "2026-06-04",
  "profile_updated": "2026-06-04"
}
```

開發過程可用這個 profile 跑測試。

---

## 設計檢查表（每次完成階段時對照）

### Agent 特質檢查
- [ ] `/check-opportunities` 沒有寫死的 step 1→2→3 流程？
- [ ] AI 會根據使用者歷史推測當前狀態？
- [ ] AI 信心 < 70% 會主動展示推理？
- [ ] 跨來源是「加分」而非「過濾條件」？

### 個人化檢查
- [ ] 同一筆機會在不同 profile 下會得到不同分數？
- [ ] `extra_keywords` 真的有被搜尋與評分用到？
- [ ] `avoid` 清單真的會把不要的機會踢掉？

### GitHub 整合檢查
- [ ] `/init-profile` 結尾有引導推 GitHub？
- [ ] `/check-opportunities` 結尾有 commit + push？
- [ ] `.gitignore` 排除 cache 和 venv？

### 時效檢查（四層）
- [ ] 層 1：搜尋關鍵字含當年度？
- [ ] 層 2：標題含舊年度的會過濾？
- [ ] 層 3：DDG 時間戳超過 3 個月的會過濾？
- [ ] 層 4：AI 在 filter 階段二次驗證？

通過全部檢查才算完成 MVP。
