# 專業版候選人推薦評分方法

## 一、背景與目的
在學術與研究審查流程中，如何快速且準確地為每個提案或論文找到合適的審查委員，是確保審查品質與效率的關鍵。本方法旨在建立一套科學、可解釋且可持續優化的候選人推薦評分機制，兼顧 **準確性 (Accuracy)**、**穩定性 (Stability)** 與 **公平性 (Fairness)**。

---



## 三、演算法評估流程（含 Bootstrap + Top-K）
### 3.1 目的

在推薦系統的情境中，常用「Top-K」來表示系統輸出的前 K 名候選人（例如 Top-10 代表系統推薦的前 10 名）。相關的評估指標（如 Recall@K、Precision@K、HitRate@K、NDCG@K）都是針對這前 K 名來衡量系統的準確度與實用性。

本演算法評估流程的設計目的在於：

- **量化平均表現**：用固定 Top-K 指標（ / HitRate@K/ NDCG@K）衡量整體準確度。
- **給出可信度**：對「題目集合」做 Bootstrap 重抽樣，不僅計算平均表現，還能估計 變異程度與 95% 信賴區間 (CI)，讓結果更具統計意義，避免只報單一數字而忽略不確定性。
（信賴區間名詞可參考：https://haosquare.com/confidence-interval/）
- **支援決策**：透過兩個指標幫助決定系統要「顯示幾名」和「召回池多大」：
  - **第一命中排名 ($K^{(hit)}$)**：代表系統第一次把正確委員排出來的位置。  
    - 例：如果 $K^{(hit)}$ 的中位數是 12，表示通常要看前 12 名才能看到第一個正確委員 → 可以用來決定介面要顯示前幾名。  
  - **Success@K 曲線**：代表在前 K 名裡至少包含一個正確委員的比例。  
    - 例：Success@10 = 65%，Success@30 = 85%，Success@50 = 95% → 可以用來決定候選名單（召回池）的寬度，要多大才夠覆蓋大部分情況。

---

### 3.2 測試資料準備（Data & Leakage Control）
- **測試集欄位**：`title, subject...`
- **姓名對齊**：建立別名表與去歧規則（姓名×機構×領域）；評測時同一人視為同一實體。
- **姓名對齊**：建立別名表與去歧規則（姓名 × 機構 × 學門），確保同一人視為同一實體，避免因姓名差異影響評測。  

- **分布檢查**：  
  - **題目字數分布**：檢查過短摘要（可能資訊不足）、過長摘要。  
  - **學門 / 領域分布**：確認不同 subject 題目的比例。    
  - **候選委員分布**：觀察 Ground Truth 覆蓋度（某些領域委員太少，可能導致評測困難）。  



---

### 3.3 指標定義
對第 $i$ 題，系統輸出前 $K$ 名 $P_i@K$，已知委員集合為 $G_i$：

命中數： $h_i(K) = |P_i@K \cap G_i|$

👉 白話：推薦的前 $K$ 人裡，有多少人真的在正確名單中。

Precision@K： $\text{Precision@K}_i = \frac{h_i(K)}{K}$

👉 白話：推薦的 $K$ 個人中，有多少比例是正確的。

HitRate@K： $\text{HitRate@K}_i = \mathbf{I}(\{h_i(K) > 0\})$

👉 白話：只要前 $K$ 人裡有至少 1 個正確的，就算成功（值=1；否則失敗（值=0）

NDCG@K：可採二元或分級相關性定義，依 ground truth 標註設計。
（可參考：https://yehjames.medium.com/python推薦系統-常見-線下-排序評估指標-90876d70a01）

Success@K 曲線：  
  $\text{Success@K} = \frac{1}{N}\sum_{i=1}^N \mathbf{I}（K^{(hit)}_i \le K)$  
  👉 白話：在所有題目裡，有多少比例能在前 $K$ 名內找到至少一個正確委員。這條曲線能幫助決定系統要顯示前幾名，才能涵蓋大部分正確答案。


#### 📌 簡單範例
假設一個題目的正確委員集合是：  
$G_i = \{A, B\}$

推薦系統輸出前 5 名：  
$P_i@5 = \{C, D, A, E, F\}$

計算結果：
- 命中數：$h_i(5) = 1$（因為 A 被命中）
- Precision@5 = $1/5 = 0.2$
- HitRate@5 = 1（有命中至少一個）

---





---


### 3.5 Bootstrap（題目層，外層）
**目的**：反映「如果題目集合不同，整體平均指標會怎麼變」，進而給 **CI**。

（可參考: https://zhuanlan.zhihu.com/p/690904510）

**步驟**
1. 設定 Bootstrap 次數為`B`與亂數種子；令測試題目數為 `N`。
2. **預先快取** 每題在各 K 的 per-query 指標（加速）。
3. 進入迴圈 `b = 1..B`：
   - 從題目索引 `1..N` **有放回地抽樣 N 次**，得到 `idx_b`。
   - 對每個 `K ∈ K_grid`，在 `idx_b` 上取該指標的宏平均：
     - `mu_ndcg[b,K]    = mean(ndcg[i,K]    for i in idx_b)`
     - `mu_hitrate[b,K] = mean(hitrate[i,K] for i in idx_b)`
     - `mu_success[b,K] = mean(1{K_hit[i] ≤ K} for i in idx_b)`
   - 另外記錄 `q50_Khit[b]`、`q90_Khit[b]`（在 `idx_b` 的 \(K^{(hit)}\) 中位數、90 分位）。
4. 最後對每個 K：
   - **各個指標基本統計量**
   - **各個指標95% CI**：`percentile(mu_recall[:,K], [2.5, 97.5])`（其餘指標同理）
---

### 3.6 專業版（可選）雙層評估：描述/關鍵字穩健性
**目的**：若同一題有多個「描述/關鍵字」變體，評估對輸出的敏感度。

- **內層（描述層）**：每題建立 `DescPool_i`（m≈10–30：標準摘要、短描述、關鍵字串）。  
  每次在題目 \(i\) 上 **有放回抽 r=5–10 個描述**，對此題先取各指標的**平均**（降低單次描述的偶然性）。
- **外層（題目層）**：同 3.5 的 Bootstrap；只是每題的值改用「該題描述層平均」。


---
### 3.4 K 值選取（K-grid）
在推薦系統評估中，單一的 $K$ 往往不足以反映不同場景的需求。因此，我們同時計算一組 **Top-K 值集合**（稱為 K-grid），例如：  
$$
K \in \{5, 10, 20, 30, 50\}
$$

這樣做的目的：
1. **小 K（如 5, 10）**：模擬「審查名單只能選很少人」的情境，要求系統在最前幾名就能給出正確人選。  
2. **大 K（如 30, 50）**：模擬「審查名單可以拉長」的情境，允許在較多候選裡面找到正確人選。  
3. **完整曲線觀察**：透過多個 K 的結果，可以畫出 Precision-Recall 曲線或 HitRate 隨 K 變化的曲線，觀察系統的「隨寬鬆度而變化的表現」。  

報告會對每個 K 分別給出：  
- 指標平均值（Precision@K、Recall@K、HitRate@K、NDCG@K）  
- 95% 信賴區間（由 Bootstrap 計算）  

---
### 3.5 分領域與總體報告
為了更細緻地分析效果，評估會同時產出：
1. **分領域 (per-subject)**：  
   - 對每個 subject / 學科領域分別計算 Precision@K、HitRate@K、NDCG@K  
   - 可看出系統在哪些領域表現好、哪些較弱  
   - 例如：醫學領域命中率 85%，工程領域命中率 70%

2. **總體 (overall)**：  
   - 對所有題目整體取宏平均，得到全域效能分數  
   - 作為系統的總體指標，便於版本比較

這樣可以同時回答：  
- 系統在「總體」上的表現如何？  
- 系統在「不同 subject」之間是否有偏差？（公平性、泛化能力）

---



### 3.9 報告格式（建議）
- **固定 K 指標**（宏平均 ± 95% CI）：  
  - Recall@10 / 20 / 30 / 50  
  - NDCG@10 / 20 / 30 / 50  
  - HitRate@K  
- **動態 K 指標**：  
  - median / p90 \(K^{(hit)}\)（含 CI）、**Success@K 曲線**（含 CI 帶）
- **延遲**：平均 / p95 Latency  
- **分群**：以領域/語言/年份/題目長度分組報同一套指標（平均 ± CI）
- **失敗分析**：命中為 0 的案例清單（含原因：在庫覆蓋、同義詞缺失、語言不匹配…）

---



### 3.11 極簡偽代碼（外層 Bootstrap）

```pseudo
INPUT: N queries, per_query metrics prec/recall/hitrate/ndcg at each K, K_grid, B
FOR b in 1..B:
  idx = sample_with_replacement(1..N, N)
  FOR K in K_grid:
    mu_ndcg[b,K]    = mean(ndcg[i,K]    for i in idx if defined)
    mu_hitrate[b,K] = mean(hitrate[i,K] for i in idx)
    mu_success[b,K] = mean( 1{K_hit[i] ≤ K} for i in idx )
  Khit_list = [K_hit[i] for i in idx if finite(K_hit[i])]
  q50_Khit[b] = quantile(Khit_list, 0.50)
  q90_Khit[b] = quantile(Khit_list, 0.90)

# 匯總（對每個 K 取均值與 95% CI）
FOR K in K_grid:
  Recall_mean[K] = mean(mu_recall[:,K])
  Recall_CI[K]   = percentile(mu_recall[:,K], [2.5, 97.5])
  # NDCG/HitRate/Success 同理
Khit_median = mean(q50_Khit); Khit_median_CI = percentile(q50_Khit, [2.5,97.5])
Khit_p90    = mean(q90_Khit); Khit_p90_CI    = percentile(q90_Khit,  [2.5,97.5])'''


---
### 4.7 產出

**(A) 指標彙總表（Overall）**
- 針對每個 K（如 5/10/20/30/50），回報下列指標的「平均值 ± 95% CI」：
  - Precision@K、Recall@K、HitRate@K、NDCG@K、Success@K
- 另回報：$K^{(hit)}$ 的 **median / p90 ± 95% CI**

**(B) 分領域表（Per-Subject）**
- 各 subject（學科/領域）分別產出與 (A) 同樣欄位，便於檢查偏科與泛化能力。
- 可附加「領域樣本數」與「權重占比」。

**(C) 圖形化結果**
- Success@K 曲線（含 95% CI 帶）：觀察 K 由小到大覆蓋率如何提升。
- 指標 vs K 折線圖（Precision/Recall/NDCG/HitRate，各自一條，含 CI）。
- $K^{(hit)}$ 分布圖（直方圖或 CDF）：觀察「第一次命中」通常落在哪個順位。

**(D) 覆蓋拆解（召回 vs 排序）**
- 候選池覆蓋率（粗召回是否把正解抓進來）。
- Top-K 命中率 / Success@K（排序是否把正解排到前面）。
- 目的：定位效能瓶頸是「召回不足」或「排序不佳」。

**(E) 可信度與可重現性**
- 報告 Bootstrap 參數：B 次數、亂數種子、K-grid、是否分層抽樣。
- 附上「平均、標準差、95% CI」三者，避免只報單一數字。
- 註明資料切分與時間切點（避免洩漏）。

**(F) 錯誤/案例分析（精簡版）**
- Top-K 失敗案例（如 Success@30=0 的題目）：列出題目、預期委員、模型前 K 結果（遮掩敏感資訊）。
- 常見失敗型態：同名歧義、冷門領域字彙、描述過短/過長等。

**(G) 決策建議**
- 依 $K^{(hit)}$ 的 **median / p90**，建議：
  - 介面預設顯示前 K（例：若 p90=40 → 預設顯示 40）。
  - 召回池寬度（例：至少召回 300，排序到前 50 供人工瀏覽）。
- 若某些 subject 顯著偏低，提出下一步改善（詞庫擴張、RRF 權重調整、召回擴寬等）。




