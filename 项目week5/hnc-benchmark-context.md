# HNC-Benchmark 项目上下文文档
# 用途：在 VSCode 中由 Claude Code 读取，继续构建工作
# 版本：v3.0 | 更新日期：2026-04-01
# 数据集：https://www.cancerimagingarchive.net/collection/hancock/
# 指南依据：NCCN Head and Neck Cancers 2026.V1

---

## 0. 项目概述

**目标**：构建一个评测专业/非专业多模态大语言模型（MLLM）在头颈癌临床推理能力的 Benchmark。

**核心设计**：
- 基于 Hancock 真实患者数据（763例，德国Erlangen单中心，2005-2019）
- 完整模拟 NCCN 标准诊疗闭环：Workup → 临床分期 → 初始治疗 → 手术 → 病理 → 辅助治疗
- 五步骤顺序推理，上下文累积，错误可级联传播
- 覆盖口腔癌、口咽癌(p16±)、下咽癌、声门喉癌、声门上喉癌 六种类型

**模型角色**：
- **被测模型（Candidate MLLM）**：接收输入，输出严格 JSON，走完五步推理链
- **评判模型（Judge LLM）**：仅评判自由文本字段（evidence、reasoning），使用预定义标准列表逐条 T/F，推荐 claude-sonnet-4-6，每字段评估2次取多数

---

## 1. Hancock 数据集关键事实

| 数据模态 | 数量 | 可用率 | 格式 | 关键说明 |
|----------|------|--------|------|----------|
| 原发灶 WSI | 709张（701例×1, 8例×2） | 92% | SVS（金字塔TIFF） | 3DHISTECH P1000, 82.44×, 0.1213µm/px |
| 淋巴结 WSI | 396张 | **52%** | SVS | 两种扫描仪，分辨率不同 |
| GeoJSON 肿瘤标注 | ~701个 | ~100% | GeoJSON | QuPath标注，多边形坐标，用于ROI提取 |
| 手术报告（英译） | >98% | >98% | .txt | DeepL翻译，来自 reports_english/ |
| 病理结构化数据 | 763例 | 100% | JSON | pathological_data.json，GT核心来源 |
| 临床结构化数据 | 763例 | 100% | JSON | clinical_data.json |
| TMA图像（IHC×8） | ~100% | ~100% | SVS/PNG | 当前版本暂不使用 |

**重要限制**：
- Hancock **不含 CT/MRI 放射影像**，`describe_focus`（影像学描述）需专家合成
- 淋巴结 WSI 缺失367例，需分类处理（见第4节）
- 部分病史/手术报告可能透露 cT/cN 信息，需专家标注 `contains_cTcN_leak: true`
- 部分手术报告含医生的辅助治疗建议，**必须在构建时删去**

**文件目录结构**：
```
HANCOCK/
├── WSI_PrimaryTumor/{oral_cavity,oropharynx,hypopharynx,larynx}/*.svs
├── WSI_LymphNode/*.svs                      # 仅396张
├── WSI_PrimaryTumor_Annotations/*.geojson   # QuPath标注
├── TMA_TumorCenter/ TMA_InvasionFront/
├── StructuredData/
│   ├── clinical_data.json
│   ├── pathological_data.json               # GT主要来源
│   └── blood_data.json
└── TextData/
    ├── reports_english/                     # Step 3 输入
    └── histories_english/                   # Step 1 输入
```

---

## 2. 专家标注需求清单

### A级：必须人工标注（无法自动化）

1. **`describe_focus`（影像学描述）**
   - Hancock 无 CT/MRI，需根据 pT+pN+tumor_site 合成影像学描述文本
   - 方法：先建 T/N 分期模板库，批量套用，专家逐例审核边界案例
   - 额外：标注 `contains_cTcN_leak: true/false`（病史/报告是否透露分期）

2. **NCCN 路径节点标注（`nccn_pathway_node`）**
   - 标注字段：`nccn_pathway_node`（如 HYPO-3）、`recommendation_category`（1类/2A类等）
   - 可半自动：规则引擎按 tumor_site + cT + cN + p16 自动定位，专家审核20%

3. **手术报告 Evidence Anchor**
   - 从手术报告中标注支持 pT/pN/R/histotype 结论的关键原文短语
   - 格式：`"pT_anchor": "resection of the larynx and right-sided piriform sinus in toto"`
   - 可 AI 辅助提取候选，专家确认

4. **指南偏差案例标注**
   - 类型A：应辅助治疗但数据显示未做 → 标注 `patient_declined: true` + `guideline_recommends`
   - 类型B：数据矛盾（如"辅助系统疗法:no"但"模式:氟尿嘧啶+顺铂"）→ 修正数据，标注正确GT

### B级：自动化+专家抽样（10-20%）

| 字段 | 自动化方法 | 审核重点 |
|------|-----------|----------|
| `preliminary_pT_stage` GT | 规则从手术报告提取 | pT3 vs pT4a 边界 |
| `staging_upgrade_occurred` | pT vs cT 自动比较 | pT < cT 异常降期案例 |
| `high_risk_gross_signals` | 关键词提取（融合/粘连/神经） | 与perinodal风险对应关系 |
| `adjuvant_decision_trigger` | 从病理特征自动生成 | 多特征并存时的优先级 |

### C级：完全自动，无需专家

- 所有 yes/no 二元字段（pathological_data.json 直接提供）
- `current_phase` GT（按步骤固定）
- 淋巴结 WSI 缺失标记（文件存在性检查）
- `p16_required`（口咽癌=true，其他=false）
- 难度分级（Simple/Medium/Hard，从分期规则自动）

---

## 3. Case 数据结构

```
Case_XXXX/
├── meta.json               # 元数据
├── step1_input.json        # Workup信息
├── step1_gt.json           # cT/cN GT
├── step2_gt.json           # NCCN路径GT
├── step3_input.json        # 手术报告+病史文本
├── step3_gt.json           # 初步pT/pN/R GT + evidence anchors
├── step4_images/
│   ├── primary_5x_scalebar.jpg    # 必须含比例尺
│   ├── primary_20x_scalebar.jpg
│   └── lymphnode_10x_scalebar.jpg # 或标记为缺失
├── step4_gt.json           # 微观病理GT（来自pathological_data.json）
└── step5_gt.json           # 辅助治疗GT
```

**meta.json 字段**：
```json
{
  "case_id": "Case_0050",
  "patient_id": "0050",
  "primary_tumor_site": "Hypopharynx",
  "difficulty": "Hard",
  "has_primary_wsi": true,
  "has_lymphnode_wsi": true,
  "ln_wsi_missing_reason": null,
  "contains_cTcN_leak": false,
  "is_non_surgical": false,
  "p16_required": false,
  "staging_upgrade": true,
  "patient_declined_adjuvant": false,
  "data_quality_flags": []
}
```

**案例集规模建议**：
- 简单（Stage I-II，无不良特征）：100例
- 中等（Stage III，1个不良特征）：200例
- 困难（Stage IV，多不良特征，含升期）：150例
- 合计核心评测集：450例

---

## 4. 淋巴结 WSI 缺失处理

| 情况 | 标记 | 被测模型应输出 | 评分规则 |
|------|------|---------------|----------|
| pN0（无转移，无需LN WSI） | `[LN_WSI_NOT_AVAILABLE — pN0]` | `perinodal_invasion: "not_applicable"` | 输出not_applicable得满分 |
| pN+但WSI不可得 | `[LN_WSI_NOT_AVAILABLE — specimen_unavailable]` | LN相关字段: `"Cannot Determine"` | Cannot Determine得满分；输出yes/no扣分+CSS惩罚 |
| `closest_margin`无法从图像测量 | Step5前系统补充提供数值 | 使用系统提供值，说明来源 | 正常评分 |

---

## 5. 五步诊疗流程规范

### Step 1：Workup → 临床分期（cT/cN）
**阶段标识**：`CLINICAL_STAGING`

**输入字段**（step1_input.json）：
- 直接提供：`year_of_initial_diagnosis`, `age_at_initial_diagnosis`, `sex`, `smoking_status`, `primarily_metastasis`, `primary_tumor_site`, `hpv_association_p16`, `icd_code`
- 专家合成：`describe_focus`（影像学描述）
- 来自Hancock：`medical_history`（histories_english/）
- **不提供**：`adjuvant_treatment_intent`（告知模型：若未给出视为no）

**被测模型必须输出**：
```json
{
  "current_phase": "CLINICAL_STAGING",
  "cT_stage": "cT3",
  "cT_stage_evidence": "引用describe_focus原文",
  "cN_stage": "cN2b",
  "cN_stage_evidence": "引用describe_focus原文",
  "next_action": "NCCN_TREATMENT_DECISION"
}
```

**阶段约束**：严禁输出任何手术、病理、辅助治疗内容

---

### Step 2：NCCN路径 → 初始治疗决策
**阶段标识**：`PRIMARY_TREATMENT_DECISION`

**输入**：Step 1 模型输出（cT/cN/肿瘤部位/p16状态）

**被测模型必须输出**：
```json
{
  "current_phase": "PRIMARY_TREATMENT_DECISION",
  "nccn_pathway_node": "HYPO-3",
  "nccn_pathway_reasoning": "从肿瘤部位→T→N→节点的完整推演",
  "recommended_treatment_options": ["放疗或同步全身治疗/放疗", "手术", "诱导化疗", "临床试验"],
  "surgery_detail": "部分或全喉下咽切除术+颈清±甲状腺切除+气管旁清扫",
  "recommendation_category": "2A类",
  "next_action": "SURGERY_EXECUTION"
}
```

**特殊规则**：
- 口咽癌 + p16未检测 → 必须给出 p16+/p16- 两种路径假设
- 非口咽癌 → 不应要求 p16 检测（否则 LLM-Judge 扣分）
- 数据集中无帕博利珠单抗治疗，需在 Prompt 中告知模型
- 非手术首选患者（`is_non_surgical: true`）→ Step 2后直接跳到 Step 5

---

### Step 3：手术报告 → 大体病理（初步）
**阶段标识**：`SURGERY_GROSS_PATHOLOGY`

**输入字段**（step3_input.json）：
- `surgery_report`（来自reports_english/，已删去医生辅助治疗建议）
- `medical_history`
- `icd_code`, `ops_codes`
- `number_of_positive_lymph_nodes`, `number_of_resected_lymph_nodes`

**零幻觉硬约束**：evidence 必须直接引用手术报告原文短语；完全未提及的特征填写 `"Not Available"`，严禁医学推断

**被测模型必须输出**（未输出不给分）：
- `histologic_type`（组织学类型）
- `preliminary_pT_stage`（用"至少pTX"表述）+ `pT_stage_evidence`
- `preliminary_resection_status`（R0/R1/R2/Uncertain）+ `resection_evidence`
- `next_action`（指向 WSI 显微病理确认）

**pN 在 Step 3 的处理**：若手术报告提到淋巴结转移，可输出 `"pN+"` 作为提示，不要求精确分期（不算分也不扣分，精确分期由 Step 4 完成）

---

### Step 4：WSI 切片 → 微观病理更新（多模态）
**阶段标识**：`WSI_MICROSCOPIC_CONFIRMATION`

**输入**：
- `primary_5x_scalebar.jpg`（全景，含比例尺）
- `primary_20x_scalebar.jpg`（细节，含比例尺）
- `lymphnode_10x_scalebar.jpg` 或缺失标记
- Step 3 模型输出摘要

**被测模型必须输出**（全部13+字段）：

更新分期：
- `updated_pT_stage`, `updated_pN_stage`, `updated_resection_status`, `updated_histologic_type`

微观侵袭特征：
- `perineural_invasion_Pn`（yes/no/Cannot Determine）
- `lymphovascular_invasion_L`（yes/no/Cannot Determine）
- `vascular_invasion_V`（yes/no/Cannot Determine）

微观形态与分级：
- `grading_hpv`（G1/G2/G3）
- `carcinoma_in_situ`（yes/no）
- `resection_status_carcinoma_in_situ`（CIS Present/CIS Absent）
- `infiltration_depth_in_mm`（必须从WSI比例尺测量）
- `closest_resection_margin_in_cm`（或注明无法测量）

淋巴结微观特征：
- `perinodal_invasion`（yes/no/Cannot Determine/not_applicable）
- `pN_stage`（精确分期）

---

### Step 5：最终病理汇总 → 辅助治疗决策
**阶段标识**：`ADJUVANT_TREATMENT_DECISION`

**输入**：Steps 1-4 的模型自身输出（累积上下文，非 GT）

**子任务A：最终病理汇总**（Step5分数的20%）
输出完整患者画像：所有临床数据 + 所有病理信息（含 Cannot Determine 字段）

**子任务B：辅助治疗决策**（Step5分数的80%，按顺序逐项输出）
```json
{
  "adjuvant_treatment_intent": "curative/palliative/no",
  "adjuvant_radiotherapy_y_n": "yes/no",
  "adjuvant_radiotherapy_type": "经皮放射治疗/近距离放射治疗",
  "adjuvant_systemic_therapy_y_n": "yes/no",
  "adjuvant_systemic_therapy_mode": "氟尿嘧啶+顺铂/顺铂/西妥昔单抗/...",
  "adjuvant_chemoradiotherapy_y_n": "yes/no",
  "reasoning": "说明基于哪条不良特征和哪个NCCN路径节点做出决定"
}
```

**升期路径切换（融贯性核心测试）**：
若 Step 4 pT > Step 1 cT（发生升期），Step 5 必须：
1. 明确说明升期及原因
2. 重新定位 NCCN 节点（如 HYPO-3 → HYPO-5）
3. 将升期列为触发辅助治疗的依据之一
→ 任何缺失触发 `[CASCADE_ERROR]` 惩罚

**辅助系统治疗模式选项**：
`氟尿嘧啶+顺铂` / `顺铂+多西他赛` / `氟尿嘧啶+卡铂` / `顺铂` / `帕博利珠单抗（数据集不存在）` / `西妥昔单抗` / `化疗（泛指）`

---

## 6. NCCN 路径映射表（Step 2 GT 规则引擎）

### 口腔癌 (Oral Cavity)
| 临床分期 | 路径节点 |
|----------|----------|
| T1-2, N0 | OR-2 |
| T3,N0 / T1-3,N1-3 / T4a,N0-3 | OR-3 |
| T4b,N0-3 / 无法切除淋巴结 | ADV-1 |
| 不适合手术 / M1 | ADV-2 |

**OR-2 手术内容**：原发灶切除±颈清 或 原发灶切除+SLN活检（pN0→end；pN+→颈清）
**OR-3 手术内容**：原发灶切除+同侧或双侧颈清（N2c需双侧）

### 口咽癌 p16阴性 (Oropharynx p16-)
| 临床分期 | 路径节点 |
|----------|----------|
| T1-2, N0-1 | ORPH-2 |
| T3-4a, N0-1 | ORPH-3 |
| T1-4a, N2-3 | ORPH-4 |
| T4b / 无法切除 | ADV-1 |
| M1 | ADV-2 |

注：NCCN 2026 中 ORPH-3/4 决策树相同，但手术细节因病情不同而有区别

**ORPH-2**：手术；RT；同步全身治疗/放疗（仅T1-2,N1，2B类）
**ORPH-3/4**：同步全身治疗/放疗；手术；诱导化疗（3类）；临床试验
**手术内容**：原发灶切除+同侧或双侧颈清

### 口咽癌 p16阳性 (Oropharynx p16+/HPV+)
| 临床分期 | 路径节点 |
|----------|----------|
| M1 | ADV-2 |
| T1-2, N0 | ORPHPV-1 |
| T0-2, N1（单节点≤3cm） | ORPHPV-2 |
| T0-2,N1（单节点>3cm或同侧≥2个≤6cm）/ T1-2,N2 / T3,N0-2 | ORPHPV-3 |
| T0-3,N3 / T4,N0-3 | ORPHPV-4 |

**ORPHPV-1**：手术；RT；临床试验
**ORPHPV-2**：手术；RT；同步全身治疗/放疗（2B类）；临床试验
**ORPHPV-3**：同步全身治疗/放疗；手术；诱导化疗（3类）；临床试验
**ORPHPV-4**：同步全身治疗/放疗（首选）；手术；诱导化疗（3类）；临床试验
**手术内容**：原发灶切除+同侧或双侧颈清（ORPHPV-1为选择性清扫）

### 下咽癌 (Hypopharynx)
| 临床分期 | 路径节点 |
|----------|----------|
| 适合喉保留（大多数T1,N0；部分T2,N0） | HYPO-2 |
| 需咽切除+全喉切除：T1-3, N0-3 | HYPO-3 |
| T4a, N0-3 | HYPO-5 |
| T4b / 无法切除 | ADV-1 |
| M1 | ADV-2 |

注：p16 检测对下咽癌无临床意义，不要求

**HYPO-2**：RT；手术；临床试验
- 手术：部分喉下咽切除（开放/内镜）+颈清±甲状腺切除+气管旁清扫

**HYPO-3**：放疗或同步全身治疗/放疗；手术；诱导化疗；临床试验（推荐等级2A类）
- 手术：部分或全喉下咽切除+颈清±甲状腺切除+气管旁清扫

**HYPO-5**：手术；诱导化疗（3类）；同步全身治疗/放疗（3类）；临床试验
- 手术：全喉下咽切除+颈清±甲状腺切除+双侧气管旁清扫

### 声门喉癌 (Glottic Larynx)
| 临床分期 | 路径节点 |
|----------|----------|
| 原位癌 / 适合保留喉功能（T1-2,N0；选定T3,N0） | GLOT-2 |
| T3需全喉切除，N0-1 | GLOT-3 |
| T3需全喉切除，N2-3 | GLOT-4 |
| T4a | GLOT-6 |
| T4b / 无法切除 | ADV-1 |
| M1 | ADV-2 |

**GLOT-2**：RT；部分喉切除（内镜或开放）+颈清（按指征）
**GLOT-3**：同步全身治疗/放疗或单纯RT；手术；诱导化疗；临床试验
- 手术：全喉切除（考虑甲状腺切除）+同侧或双侧颈清
**GLOT-4**：同步全身治疗/放疗；手术；诱导化疗；临床试验
- 手术：全喉切除+甲状腺切除+颈清+气管旁清扫
**GLOT-6**：手术（T4a首选）；拒绝则考虑放化疗/临床试验/诱导化疗
- 手术：全喉切除+颈清+甲状腺切除（喉外侵犯/声门下扩展时）

### 声门上喉癌 (Supraglottic Larynx)
| 临床分期 | 路径节点 |
|----------|----------|
| 适合保留喉功能（大多T1-2,N0；部分T3） | SUPRA-2 |
| 需全喉切除，T3,N0 | SUPRA-3 |
| T4a, N0 | SUPRA-8 |
| 淋巴结阳性 → SUPRA-4分流 | |
| → 适合保留喉（T1-2,N+；部分T3,N1） | SUPRA-5 |
| → 需全喉切除（大多T3,N1-3） | SUPRA-6 |
| → T4a, N1-3 | SUPRA-8 |
| T4b / 无法切除 | ADV-1 |
| M1 | ADV-2 |

**SUPRA-2**：手术；RT
- 手术：内镜或开放部分喉切除+颈清
**SUPRA-3**：同步全身治疗/放疗或单纯RT；手术；诱导化疗；临床试验
- 手术：全喉切除+甲状腺切除+颈清
**SUPRA-5**：同步放化疗或RT；手术；诱导化疗；临床试验
- 手术：内镜或开放部分喉切除+颈清
**SUPRA-6**：同步全身治疗/放疗；手术；诱导化疗；临床试验
- 手术：全喉切除+同侧甲状腺切除+颈清
**SUPRA-8**：手术（T4a首选）；拒绝则考虑放化疗/临床试验/诱导化疗
- 手术：内镜或开放部分喉切除+颈部淋巴结清扫

---

## 7. 辅助治疗决策树（Step 5 GT 基础）

### 不良病理特征分级

| 级别 | 特征 | 结果 | 推荐等级 |
|------|------|------|----------|
| 🔴 高危 | 淋巴结外侵犯(perinodal=yes) 和/或 切缘阳性(R1 / margin<0.1cm) | 系统治疗+放疗（同步放化疗） | **1类** |
| 🟡 其他风险 | pT3/T4、pN2/N3、多淋巴结阳性、LVI+、Pn+、单淋巴结阳性无其他特征 | 放疗 或 考虑系统治疗/放疗 | 2A类 |
| 🟢 无不良特征 | pN0 或 pN1且无其他风险 | 观察(end) 或 考虑RT | — |

### 各类型辅助治疗决策流

**口腔癌 OR-2 术后**：
- 无阳性淋巴结，无不良特征 → end
- 1个阳性淋巴结，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗（1类）
- 切缘阳性（无ENE）→ 可行则再次切除；切缘阴性后考虑 RT

**口腔癌 OR-3 术后**：
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗（1类）
- 切缘阳性（无ENE）→ 系统治疗/放疗（1类）或再次切除（若可行，切缘阴性后考虑RT）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**口咽癌 p16- ORPH-2 术后**：
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗（1类）
- 切缘阳性 → 再次切除 or 放疗 or 考虑系统治疗/放疗
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**口咽癌 p16- ORPH-3/4 术后**：
- 无不良特征 → RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**口咽癌 p16+ ORPHPV 各节点术后**：
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**下咽癌 HYPO-2/3 术后**：
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**下咽癌 HYPO-5 术后（pT4a升期后切入）**：
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**声门喉癌 GLOT-3/4/6 术后**：
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**声门上喉癌 各节点术后（SUPRA-2/3/5/6/8）**：
- 淋巴结阴性（T1-2,N0）→ end
- 单个淋巴结阳性，无其他不良特征 → 考虑 RT
- 淋巴结阳性；切缘阳性 → 再次切除（高度筛选可行者）or RT or 考虑系统治疗/放疗
- 🔴 淋巴结外侵犯 → 系统治疗/放疗（1类）or 放疗（2B类，特定患者）
- 淋巴结阳性+其他不良特征 → 放疗 or 考虑系统治疗/放疗
- 淋巴结阴性（T3-T4a,N0）→ 进入对应SUPRA-3/8辅助决策

---

## 8. 评分体系

### 总分结构
```
步骤权重：Step1=15%, Step2=20%, Step3=20%, Step4=25%, Step5=20%
每步分数：60% × 结构化字段分 + 40% × LLM-Judge分
Step5分数：20% × 汇总子任务 + 80% × 决策子任务
Case总分：Σ(权重 × 步骤分) × safety_multiplier
safety_multiplier：max(0.5, 1.0 - critical_error_count × 0.2)
双轨运行：AR Score（级联传播）和 IND Score（每步注入GT）分别报告
```

### 结构化字段评分规则

| 字段类型 | 满分=1.0 | 部分分=0.5 | 零分=0 |
|----------|---------|-----------|--------|
| 分期（cT/pT/cN/pN） | 精确匹配 | 相邻分期（如cT2↔cT3） | 跨越两级 |
| NCCN节点 | 精确匹配（如HYPO-3） | 同部位相邻节点 | 不同部位节点 |
| 推荐等级 | 精确（1类/2A/2B/3类） | 1类↔2A（方向对） | 1类↔3类 |
| 二元yes/no（🔴高危字段） | 精确；Cannot Determine（无WSI时）=满分 | — | 反向（触发CSS惩罚） |
| R0/R1/R2 | 精确 | R0↔R1=0.3 | R0↔R2 |
| 数值（DOI, margin） | 误差≤0.05cm/1mm | 误差0.05~0.1cm/1~3mm | 超出范围 |
| 辅助治疗yes/no（🔴决策字段） | 精确 | — | 反向（触发CSS惩罚） |

### LLM-Judge 评分（文本字段，每字段3条标准逐一T/F）
通用3条标准：
1. 内容基于提供的原始信息，未超出信息范围（无幻觉）
2. 逻辑可追溯——结论与引用依据存在明确因果关系
3. 领域术语使用准确，无明显病理/临床概念错误

Step 5 决策字段额外第4条：
4. 正确识别了高危特征并映射到指南对应处理路径

### 临床安全惩罚（独立 CSS 分，不进总分）

| 错误 | Flag | 惩罚 |
|------|------|------|
| 漏报perinodal | [PERINODAL_MISS] | Step4分×0.3；CSS-0.30 |
| 漏开放化疗 | [CHEMORT_MISSED] | Step5决策分×0.2；CSS-0.30 |
| 阶段时序混淆 | [TEMPORAL_VIOLATION] | 违规字段得分清零 |
| 幻觉引用 | [HALLUCINATION_FLAG] | 该evidence字段清零；CSS-0.10 |
| 无WSI时输出yes/no | [LN_WSI_HALLUC] | 该字段×0.2；CSS-0.10 |
| 升期后路径未切换 | [CASCADE_ERROR] | 额外-0.15 |

```
CSS = 1.0 - (高危漏报数 × 0.30) - (幻觉次数 × 0.10)
```

---

## 9. 特殊案例处理规则

| 情况 | meta.json标记 | 处理方式 |
|------|--------------|----------|
| 非手术首选（放化疗） | `is_non_surgical: true` | Step 2后直接跳到Step 5，Step 3/4标记NA |
| 口咽癌p16未检测 | `p16_required: true` | Step 2必须给出p16+/p16-两种路径 |
| 非口咽癌要求p16 | — | LLM-Judge第③条评为False，扣分 |
| 病史透露cTcN | `contains_cTcN_leak: true` | 作为"信息更新能力"困难子集 |
| 患者拒绝辅助治疗 | `patient_declined: true` | GT补充`guideline_recommends_adjuvant`，按指南建议评分 |
| 数据矛盾患者 | `data_contradiction: true` | 专家修正后提供正确GT |

---

## 10. 待完成工作（TODO）

- [ ] **describe_focus 模板库**：按肿瘤部位 × T分期 × N分期建立标准影像描述模板
- [ ] **NCCN节点规则引擎**：代码实现 tumor_site + cT + cN + p16 → pathway_node 的自动映射
- [ ] **手术报告 Evidence Anchor 提取**：AI辅助 + 专家确认，建立 anchor 标注库
- [ ] **指南偏差案例筛选**：从763例中筛选出A/B类型患者，专家标注修正
- [ ] **WSI Patch 提取脚本**：基于GeoJSON标注，用OpenSlide提取5x/20x/10x patches并添加比例尺
- [ ] **评分器实现**：结构化字段规则评分器 + LLM-Judge调用封装
- [ ] **案例集分层**：按难度筛选450例核心评测集（100简+200中+150难）
- [ ] **Prompt模板最终版**：五步Prompt含所有约束规则和格式要求

---

*文档结束 | 下次继续时请将此文件提供给 Claude，说明当前进展到哪个 TODO 项*
