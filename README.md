你是機構工程師，負責開發一個玻璃強度分析 Web App（Streamlit）。
請依照以下規格建立完整的專案：

## 專案名稱
glass-strength-analyzer

## 技術棧
- Frontend: Streamlit
- Backend: Python 3.10+
- 3D 解析: python-occ (pythonocc-core) 或 regex-based STEP parser
- 視覺化: matplotlib (3D), plotly
- 報告: python-pptx
- 標準庫: 內建 MIL-STD-810G Method 516 & IK01~IK10 查表

## 目錄結構
glass-strength-analyzer/
├── app.py                  # Streamlit 主程式
├── core/
│   ├── step_parser.py      # STEP 幾何解析（尺寸/R角/孔位/厚度）
│   ├── materials.py        # 材料資料庫（玻璃/PC/PMMA）
│   ├── bonding.py          # OCA vs Air Bonding 複合板模型
│   ├── standards.py        # MIL-810G Method516 + IK01~IK10 查表
│   ├── physics.py          # 板彎理論+Hertz接觸+能量法+Nadai應力
│   └── recommender.py      # 自動迭代設計建議（目標SF≥1.5）
├── viz/
│   ├── stress_3d.py        # matplotlib 3D 應力圖（5點落球動畫）
│   └── layout_2d.py        # 落球測試佈局俯視圖
├── report/
│   └── report_gen.py       # python-pptx 報告產生（含截圖）
├── data/
│   └── materials.json      # 材料庫 JSON
├── requirements.txt
└── README.md

## 核心功能規格

### 1. STEP Parser (step_parser.py)
從 STEP 檔案解析：
- 玻璃面板：頂面尺寸 (a × b mm)、厚度 t、四角 R 角、邊緣 fillet R
- 前框：外框尺寸、材質 bounding box
- Panel 辨識：若存在 panel layer，回傳厚度與位置

### 2. 材料庫 (materials.json)
至少包含：
- 玻璃: Generic CSG (E=70GPa, ν=0.23, MOR=600MPa)
       Gorilla Glass 6 (MOR=800MPa)
       Dragontrail X (MOR=900MPa)
       Sapphire (MOR=1900MPa)
- 前框: PC Lexan EXL9330 (E=2350MPa, σ_flex=87MPa)
        PC+GF30 (E=7000MPa)
        Al 6061-T6 (E=69GPa, σ_y=276MPa)
- OCA: 厚度 0.175mm, E=0.1MPa (彈性緩衝)
- Touch Panel: 0.7mm 玻璃基板

### 3. Bonding Model (bonding.py)
- Air Bonding: 只計算外層玻璃，D = E_glass × t_glass³ / 12(1-ν²)
- OCA 貼合: 複合截面 D_eff = Σ(EᵢIᵢ + EᵢAᵢdᵢ²) / (1-ν²)
  需輸出等效厚度 t_eff 供後續計算

### 4. 落球測試標準 (standards.py)
- 手動輸入: 質量(g) + 高度(mm)
- MIL-STD-810G Method 516.6:
  Procedure I (Transit Drop): 依重量查表 (< 9kg → 1.22m, 9~22kg → 0.91m...)
  Procedure IV (Operational Drop): 0.91m, 各面/邊/角
- IK 等級 (IEC 62262):
  IK01=0.14J, IK02=0.2J, IK03=0.35J, IK04=0.5J, IK05=0.7J
  IK06=1J, IK07=2J, IK08=5J, IK09=10J, IK10=20J
  錘頭形狀: IK01~06=半球R10, IK07~10=半球R25
  由能量 Ek 反推: m×g×h = Ek → 預設 h=0.3m 求 m

### 5. 物理引擎 (physics.py)
實作以下計算流程：
a) 板剛度: D = E_eff × t_eff³ / [12(1-ν²)]
b) 彈簧常數: k(x,y) = D / [α(x/a, y/b) × b²]  (雙重Fourier級數 N=50)
c) 動能: Ek = ½mv² = m×g×h
d) 峰值衝擊力: F = √(2k × Ek)
e) Hertz接觸半徑: rc = [3FR/(4E*)]^(1/3)
f) Nadai最大應力: σ = 3F(1+ν)/(2πt²) × ln(b_eff/rc)
g) 邊緣應力: σ_edge = Kt × σ, Kt = 1 + 0.65√(t/2ρ)
h) 安全係數: SF = MOR / σ_max

### 6. 推薦引擎 (recommender.py)
若 SF < 1.5，自動建議：
- 最小需求厚度（迭代至 SF≥1.5）
- 材質升級建議（按成本排序）
- 邊緣 R 角改善（Kt 降低效果）
- 估算設計變更對成本的影響

### 7. Streamlit UI (app.py)
分為 4 個頁籤：
Tab 1 "上傳設計" → STEP 上傳 + 幾何預覽 + 材質選擇 + 貼合方式
Tab 2 "測試設定" → 落球參數 / MIL-810G / IK 等級選擇 + 5點佈局設定
Tab 3 "分析結果" → SF 總表 + 3D 應力圖 + 落球動畫 + Pass/Fail badge
Tab 4 "報告輸出" → 設計變更建議 + 下載 PPTX / PDF

### 8. 3D 視覺化 (stress_3d.py)
- 使用 matplotlib 3D + Streamlit st.pyplot()
- 顯示玻璃板 surface mesh (60×60)
- 應力分布：colormap DBEAFE→FCA5A5→EF4444
- 落球點：scatter marker，active 點高亮
- 支援動畫：返回 frame list 供 st.image() 播放
- 背景: #EEF2F7（淺藍灰）

## UI 設計風格
- 主色: Navy #1A2B4A, Accent Red #C0392B, Gold #D4891A
- Pass: Green #166534 背景 #DCFCE7
- Fail: Red #C0392B 背景 #FEE2E2
- 使用 st.metric() 顯示 SF 值
- 卡片式佈局（st.container + custom CSS）

## 額外要求
1. 所有計算結果需附公式說明（可摺疊展開）
2. 材料庫可由使用者上傳 CSV 新增自定義材料
3. 支援繁體中文介面，英文公式標記
4. 計算完成後自動生成 5 點測試佈局圖（含安全距離標示）
5. requirements.txt 包含所有相依套件版本

請先建立目錄結構與所有檔案，然後依序實作各模組，
最後確認 `streamlit run app.py` 可正常啟動並顯示完整 UI。
