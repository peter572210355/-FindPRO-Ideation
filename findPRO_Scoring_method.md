# 專業版候選人推薦評分方法

## 一、背景與目的
在學術與研究審查流程中，如何快速且準確地為每個提案或論文找到合適的審查委員，是確保審查品質與效率的關鍵。本方法旨在建立一套科學、可解釋且可持續優化的候選人推薦評分機制，兼顧 **準確性 (Accuracy)**、**穩定性 (Stability)** 與 **公平性 (Fairness)**。

---

## 二、評分方法架構


## 三、演算法評估流程（含 Bootstrap + Top-K）

### 3.1 目的（Why）
- **量化平均表現**：用固定 Top-K 指標（Recall@K / NDCG@K / HitRate@K）衡量整體準確度。
- **給出可信度**：對「題目集合」做 **Bootstrap** 重抽樣，取得 **95% 信賴區間**（CI），避免只報單一數字。
- **支援決策**：用 **K^{(hit)}**（第一命中排名）與 **Success@K** 曲線，回答「介面要顯示前幾名？」與「召回池要多寬？」。

---

### 3.2 資料準備（Data & Leakage Control）
- **測試集欄位**：`title, subject...`
- **姓名對齊**：建立別名表與去歧規則（姓名×機構×領域）；評測時同一人視為同一實體。


---

### 3.3 指標定義（Per-Query → Aggregate）
對第 $i$ 題，系統輸出前 $K$ 名 $P_i@K$，已知委員集合為 $G_i$：

命中數： $h_i(K) = |P_i@K \cap G_i|$

Precision@K： $\text{Precision@K}_i = \frac{h_i(K)}{K}$

HitRate@K： $\text{HitRate@K}_i = \mathbf{1}\{h_i(K) > 0\}$

NDCG@K：可採二元或分級相關性定義，依 ground truth 標註設計。
（可參考：https://yehjames.medium.com/python推薦系統-常見-線下-排序評估指標-90876d70a01）

第一命中排名： $K^{(hit)}_i = \min\{k: h_i(k) > 0\}, \quad \text{若無命中則記為 } \infty$
#### 📌 簡單範例
假設一個題目的正確委員集合是：  
$G_i = \{A, B\}$

推薦系統輸出前 5 名：  
$P_i@5 = \{C, D, A, E, F\}$

計算結果：
- 命中數：$h_i(5) = 1$（因為 A 被命中）
- Precision@5 = $1/5 = 0.2$
- HitRate@5 = 1（有命中至少一個）
- 第一命中排名 = 3（因為第 3 個推薦才命中 A）
- NDCG@5：因為 A 在第 3 位，得分比放在第 1 位要低，但比放在第 5 位高。

---

#### 整體彙總（宏平均）

對所有題目的 per-query 指標取平均；同時計算 **Success@K**：

$$
\text{Success@K} = \frac{1}{N}\sum_{i=1}^{N} \mathbf{1}\{K^{(hit)}_i \le K\}
$$

並統計 $K^{(hit)}$ 的 **median / p90**，做為 Top-K 決策依據。



---

### 3.4 K 網格（K-grid）
設定同時計算的 K 值集合，例如：
報告會對每個 K 各自給出「平均值 ± 95% CI」。

---

### 3.5 Bootstrap（題目層，外層）
**目的**：反映「如果題目集合不同，整體平均指標會怎麼變」，進而給 **CI**。

**步驟**
1. 設定 `B`（例如 1000）與亂數種子；令測試題目數為 `N`。
2. **預先快取** 每題在各 K 的 per-query 指標（加速）。
3. 進入迴圈 `b = 1..B`：
   - 從題目索引 `1..N` **有放回地抽樣 N 次**，得到 `idx_b`。
   - 對每個 `K ∈ K_grid`，在 `idx_b` 上取該指標的宏平均：
     - `mu_recall[b,K]  = mean(recall[i,K]  for i in idx_b)`
     - `mu_ndcg[b,K]    = mean(ndcg[i,K]    for i in idx_b)`
     - `mu_hitrate[b,K] = mean(hitrate[i,K] for i in idx_b)`
     - `mu_success[b,K] = mean(1{K_hit[i] ≤ K} for i in idx_b)`
   - 另外記錄 `q50_Khit[b]`、`q90_Khit[b]`（在 `idx_b` 的 \(K^{(hit)}\) 中位數、90 分位）。
4. 最後對每個 K：
   - **平均值**：`mean(mu_recall[:,K])`
   - **95% CI**：`percentile(mu_recall[:,K], [2.5, 97.5])`（其餘指標同理）
   - \(K^{(hit)}\) 的 **median/p90** 也以百分位法給 CI。

> 若題目分布不均（領域/語言），改用 **分層 Bootstrap**：在各分層內按比例有放回抽樣，再合併。

---

### 3.6 專業版（可選）雙層評估：描述/關鍵字穩健性
**目的**：若同一題有多個「描述/關鍵字」變體，評估對輸出的敏感度。

- **內層（描述層）**：每題建立 `DescPool_i`（m≈10–30：標準摘要、短描述、關鍵字串）。  
  每次在題目 \(i\) 上 **有放回抽 r=5–10 個描述**，對此題先取各指標的**平均**（降低單次描述的偶然性）。
- **外層（題目層）**：同 3.5 的 Bootstrap；只是每題的值改用「該題描述層平均」。

> 若同時用 BM25 與向量檢索，可對每個描述先做 **RRF 集成** 再計算 per-query 指標。

---

### 3.7 成對比較（A/B）——顯著性檢驗
**目的**：比較新舊兩版是否「真的更好」，而非樣本偶然。

- 在每次外層抽樣 `idx_b` 上，分別算 A、B 的宏平均（同一批題目）。
- 差值 `Δ_b = μ_A(b,K) − μ_B(b,K)`；對 `{Δ_b}` 取 95% CI。  
- **CI 不含 0** ⇒ 差異顯著。

---

### 3.8 覆蓋拆解（召回 vs 排序）
同時報：
- **候選池覆蓋率**：Ground Truth 是否至少出現在「大候選池」（粗召回）內 → **召回問題**。
- **Top-K 命中率 / Success@K**：已在候選池內但未排進前 K → **排序問題**。

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

### 3.10 參數建議與早停
- `K_grid = [10, 20, 30, 50]`  
- `B = 1000`（預檢 300–500）  
- 雙層時：每題描述池 `m = 10–30`、內層每次抽 `r = 5–10`  
- **早停**：若核心指標（如 Recall@30）之 95% CI 寬度 < 0.02 且最近 200 次變化 < 0.005，即可停止。

---

### 3.11 極簡偽代碼（外層 Bootstrap）

```pseudo
INPUT: N queries, per_query metrics prec/recall/hitrate/ndcg at each K, K_grid, B
FOR b in 1..B:
  idx = sample_with_replacement(1..N, N)
  FOR K in K_grid:
    mu_recall[b,K]  = mean(recall[i,K]  for i in idx if defined)
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
Khit_p90    = mean(q90_Khit); Khit_p90_CI    = percentile(q90_Khit,  [2.5,97.5])



## 四、專業版評估流程（多描述 × Bootstrap × Top-K）

### 4.1 目的
- **量平均表現**：用固定的 Top-K 指標（Precision@K / Recall@K / HitRate@K / NDCG@K）衡量排序品質。  
- **量不確定性**：對題目集合做 **Bootstrap**，給出 95% 信賴區間（CI），而不是只報單一數字。  
- **量穩健性**：同一題目會有多個描述/關鍵字版本；內層隨機抽描述後取平均，檢驗系統對描述寫法的敏感度。  
- **支援決策**：利用第一命中排名 $K^{(hit)}$ 與 **Success@K 曲線**，幫助決定 UI 要顯示幾名候選人、候選池要多寬。  

---

### 4.2 輸入與資料準備
- **測試題目集**：`qid, title, (optional) abstract/desc, keywords, year, field, gt_reviewers[]`
- **描述池（每題）**：從 *title*、*摘要精簡句*、*官方/作者關鍵字*、*關鍵字擴寫* 產生，每題約 `m ≈ 10–30` 條。  
- **時間切分**：候選人索引僅用 cutoff 之前資料，測試題目用 cutoff 之後資料（避免洩漏）。  
- **姓名對齊 / COI**：處理同名異寫（姓名×機構×領域），可選擇有/無 COI 過濾各算一份。  

---

### 4.3 指標定義（Per-Query → Aggregate）

對第 $i$ 題，系統輸出前 $K$ 名 $P_i@K$，已知委員集合為 $G_i$：

- **命中數**  
  $$
  h_i(K) = |P_i@K \cap G_i|
  $$

- **Precision@K**  
  $$
  \text{Precision@K}_i = \frac{h_i(K)}{K}
  $$

- **Recall@K**  
  $$
  \text{Recall@K}_i = \frac{h_i(K)}{|G_i|}
  $$
  若 $|G_i|=0$，此題不納入平均，另行統計。

- **HitRate@K**  
  $$
  \text{HitRate@K}_i = \mathbf{1}\{h_i(K) > 0\}
  $$

- **NDCG@K**  
  可採二元或分級相關性定義，依 ground truth 標註設計。

- **第一命中排名 $K^{(hit)}_i$**  
  $$
  K^{(hit)}_i = \min\{k: h_i(k) > 0\}, \quad \text{若無命中則記為 } \infty
  $$

**整體彙總（宏平均）**  
對所有題目的 per-query 指標取平均；同時計算 **Success@K**：
$$
\text{Success@K} = \frac{1}{N}\sum_{i=1}^{N} \mathbf{1}\{K^{(hit)}_i \le K\}
$$
並統計 $K^{(hit)}$ 的 **median / p90**，做為 Top-K 決策依據。

---

### 4.4 K 網格（K-grid）
一次同時計算多個 K 值，例如 `K = {10, 20, 30, 50}`。  
每個 K 都提供 **均值 ± 95% CI**，並繪製 **Success@K 曲線**。  

---

### 4.5 雙層 Bootstrap 設計

- **內層：描述層穩健平均**  
  - 每題從描述池 `DescPool_i` 有放回抽取 $r$ 個描述。  
  - 每個描述各自檢索 → 算 per-query 指標 → 取平均。  
  - 得到該題在本次抽樣的穩健分數。

- **外層：題目層 Bootstrap**  
  - 測試集有 $N$ 題；每次有放回抽樣 $N$ 題形成一批。  
  - 在這批題目上，計算每個 K 的宏平均指標。  
  - 重複 $B=500\sim1000$ 次，得到分布 → 取 95% CI。  
  - 同時計算 $K^{(hit)}$ 的 median / p90 ± CI。

---

### 4.6 評估流程步驟
1. **設定**  
   - $K = \{10, 20, 30, 50\}$  
   - 內層 $r = 5\sim10$；外層 $B = 500\sim1000$  

2. **預先快取**  
   - 每個（題目 × 描述）先算檢索結果與 per-query 指標。  
   - Bootstrap 階段只需做抽樣與平均 → 加速。  

3. **外層迴圈**  
   - 每次抽 $N$ 題；對每題再抽 $r$ 描述並取平均。  
   - 計算 Precision/Recall/HitRate/NDCG/Success@K。  
   - 記錄 $K^{(hit)}$ 的 median / p90。  

4. **匯總**  
   - 報告各 K 的 **均值 ± 95% CI**。  
   - 報告 $K^{(hit)}$ 的 median / p90 ± CI。  
   - 畫 **Success@K 曲線**與各指標折線圖。  

---

### 4.7 產出
- **表格**：Precision@K, Recall@K, NDCG@K, HitRate@K, Success@K（均值 ± CI）  
- **圖表**：Success@K 曲線（帶 CI）、各指標 vs K 折線圖  
- **診斷**：候選池覆蓋率 vs Top-K 命中率  

---

### 4.8 總結價值
- **更準確**：不只算平均，還有信賴區間。  
- **更穩健**：多描述隨機檢驗，避免因表述差異導致評估偏差。  
- **更實用**：$K^{(hit)}$ median / p90 + Success@K 曲線 → 幫助設定系統的預設顯示長度。  
- **可擴充**：支援基線 vs 新模型比較，做顯著性檢驗。  
