# HNC-Benchmark 项目上下文文档
# 版本：v5.0 | 更新：2026-04-03（Step 1/2 合并，删除 describe_focus 合成）
# 数据集：https://www.cancerimagingarchive.net/collection/hancock/
# 指南：NCCN Head and Neck Cancers 2026.V1
# 状态：设计阶段完成，进入初步搭建阶段

---

## 0. 项目概述

**目标**：构建评测专业/非专业多模态大语言模型（MLLM）在头颈癌临床推理能力的 Benchmark，并做成公开网站（支持本地部署）。

**核心设计**：
- 基于 Hancock 真实患者数据（763例，德国Erlangen单中心，2005-2019）
- 完整模拟 NCCN 标准诊疗闭环：临床分期决策 → 初始治疗决策 → 手术 → 病理 → 辅助治疗
- **四步**顺序推理（原 Step 1/2 合并为 Step 1），上下文累积，错误可级联传播
- 覆盖：口腔癌、口咽癌(p16-)、口咽癌(p16+)、下咽癌、声门喉癌、声门上喉癌

**两种模型角色**：
- **被测模型（Candidate MLLM）**：接收输入，输出严格 JSON，走完四步推理链
- **评判模型（Judge LLM）**：评判自由文本字段（reasoning），用预定义标准逐条 T/F

---

## 1. Hancock 数据集关键事实

| 模态 | 数量 | 可用率 | 格式 | 说明 |
|------|------|--------|------|------|
| 原发灶 WSI | 709张（701例×1，8例×2） | 92% | SVS | 3DHISTECH P1000，82.44×，0.1213µm/px |
| 淋巴结 WSI | 396张 | **52%** | SVS | 两种扫描仪，分辨率不同 |
| GeoJSON 肿瘤标注 | ~701个 | ~100% | GeoJSON | QuPath标注，多边形坐标，用于ROI提取 |
| 手术报告（英译） | >98% | >98% | .txt | DeepL翻译，reports_english/ |
| 病理结构化数据 | 763例 | 100% | JSON | pathological_data.json，GT核心来源 |
| 临床结构化数据 | 763例 | 100% | JSON | clinical_data.json |

**重要限制与预处理注意**：
- 淋巴结 WSI 缺失 367 例，需分类处理（见第4节）
- Hancock 将声门喉癌与声门上喉癌**均归为喉癌**，需额外根据 ICD codes 区分，由 大模型 归类后专家检查
- 部分病史/手术报告可能透露 cT/cN 信息，需标注 `cont大模型ns_cTcN_leak: true` 并**删去泄露内容**
- 手术报告中医生直接给出的辅助治疗建议**必须删去**
- 531号患者为口咽癌但 p16 未测试，且整体 ground truth 较少，**建议剔除**
- 20%患者由于癌症初期或手术原因无pN记录（记录为NX），**建议根据情况进行归类为pN0或划分为数据部分缺失的高难度题**

**文件目录**：
```
HANCOCK/
├── WSI_PrimaryTumor/{oral_cavity,oropharynx,hypopharynx,larynx}/*.svs
├── WSI_LymphNode/*.svs
├── WSI_PrimaryTumor_Annotations/*.geojson
├── StructuredData/
│   ├── clinical_data.json
│   ├── pathological_data.json     ← GT主要来源
│   └── blood_data.json
│   └── blood_data_reference_ranges.json
└── TextData/
    ├── reports_english/           ← Step 2 输入
    └── histories_english/         ← Step 1 辅助输入
```

---

## 2. 专家标注需求清单（v5.0）

### A级：必须介入（可大模型辅助生成初稿，专家检查完善）

**① NCCN 路径节点（`nccn_pathway_node`）— Step 1 GT**
- 规则引擎按 tumor_site + cT + cN + p16 自动定位节点，专家抽样验证20%
- 重点审核：T3/T4a 边界、声门上喉癌 N+ 分流
- **注意**：部分节点推荐等级不存在（如 OR-2、OR-3 无固定等级），不可自行填充，见第6节映射表

**② 手术报告 Evidence Anchor**
- 从手术报告中标注支持 pT/pN/R/histotype 结论的关键原文短语
- 格式：`"pT_anchor": "resection of the larynx and right-sided piriform sinus in toto"`
- 大模型 辅助提取候选(没有则标注为NULL)，需专家确认

**③指南偏差案例（两类）**
- **类型A**：指南应辅助治疗但数据显示患者未做 → 标注 `patient_declined: true` + `guideline_recommends`
- **类型B**：数据矛盾（如"辅助系统疗法:no" 但"模式:氟尿嘧啶+顺铂"）→ 修正矛盾，标注正确GT
- 可 大模型 辅助筛选出这两类，专家最终确认

### B级：自动化+可专家抽样（10-20%）

| 字段 | 自动化方法 | 审核重点 |
|------|-----------|----------|
| `preliminary_pT_stage` GT | 规则从手术报告提取 | pT3 vs pT4a 边界 |
| `staging_upgrade_occurred` | pT vs cT 自动比较 | pT < cT 异常降期案例 |
| `high_risk_gross_signals` | 大模型辅助关键词提取 | 与 perinodal 对应关系 |
| `adjuvant_decision_trigger` | 从病理特征自动生成 | 多特征并存优先级 |
| ICD codes 分类（声门/声门上） | 大模型归类 | 全量专家检查 |
| 口咽癌 p16 状态处理 | 大模型归类 | hpv_association_p16 |

注1:对所有患者**均不在输入中提供** `hpv_association_p16` 字段的阴/阳性结果
- Step 1 内设附加子任务：在询问 NCCN 路径时，口咽癌患者被测模型**必须先声明"需要 p16 结果才能给出路径"**，然后系统提供 p16 结果，模型再给出最终路径
- 此附加子任务需单独设计评分逻辑，独立报告
- **例外**：531号口咽癌患者未测试 p16 且 ground truth 较少，**建议直接剔除**
 
注2:声门喉癌 vs 声门上喉癌区分**
- 原数据集将两者均归为"喉癌"
- 映射时根据 ICD codes 由 大模型 自动归类，专家全量检查确认
- 这两类病种使用完全不同的 NCCN 路径（GLOT vs SUPRA），区分正确与否直接影响 GT


### C级：完全自动

- 所有 yes/no 二元字段（pathological_data.json 直接提供）
- `current_phase` GT（按步骤固定）
- 淋巴结 WSI 缺失标记（文件存在性检查）
- `p16_required`（口咽癌=true，其他=false）
- 难度分级（Simple/Medium/Hard）

---

## 3. Case 数据结构

```
Case_XXXX/
├── meta.json
├── step1_input.json        # 患者基本信息 + 直接给出的 cT/cN（不含p16结果）
├── step1_gt.json           # NCCN路径GT + p16子任务GT
├── step2_input.json        # 手术报告+病史（已删辅助治疗建议）
├── step2_gt.json           # 初步pT/pN/R GT + evidence anchors
├── step3_images/
│   ├── primary_5x_scalebar.jpg
│   ├── primary_20x_scalebar.jpg
│   └── lymphnode_10x_scalebar.jpg  # 或缺失标记
├── step3_gt.json           # 微观病理GT（pathological_data.json）
└── step4_gt.json           # 辅助治疗GT（含指南偏差修正）
```

**meta.json 字段**：
```json
{
  "case_id": "Case_0050",
  "patient_id": "0050",
  "primary_tumor_site": "Hypopharynx|Oral_cavity|Larynx|Oropharynx",
  "cancer_subtype": "glottic|supraglottic|hypopharynx|oral_cavity|oropharynx_p16neg|oropharynx_p16pos",
  "difficulty": "Simple|Medium|Hard",
  "has_primary_wsi": true|false,
  "has_lymphnode_wsi": true|false,
  "ln_wsi_missing_reason": null,
  "contains_cTcN_leak":  true|false,
  "is_non_surgical": true|false,
  "p16_required":  true|false,
  "staging_change":  true|false,
  "patient_declined_adjuvant": true|false,
  "data_contradiction": true|false,
  "icd_codes_larynx_subtype": "glottic|supraglottic|null",
  "data_quality_flags": []
}
```

**案例集规模（视情况而定）**：
- 简单（Stage I-II，无不良特征）：100例
- 中等（Stage III，1个不良特征）：200例
- 困难（Stage IV，多不良特征，含升期）：150例
- 合计核心评测集：约 450 例

---

## 4. 淋巴结 WSI 缺失处理

| 情况 | 标记 | 模型应输出 | 评分 |
|------|------|-----------|------|
| pN0（无转移） | `[LN_WSI_NOT_AV大模型LABLE — pN0]` | `"not_applicable"` | 输出not_applicable得满分 |
| pN+但WSI不可得 | `[LN_WSI_NOT_AV大模型LABLE — specimen_unav大模型lable]` | `"Cannot Determine"` | Cannot Determine满分；输出yes/no扣分+CSS惩罚 |


---

## 5. 四步诊疗流程规范

### Step 1：临床分期 + NCCN路径 → 初始治疗决策（合并步骤）
**阶段**：`STAGING_AND_TREATMENT_DECISION`

**输入**（step1_input.json）：
- `year_of_initial_diagnosis`, `age_at_initial_diagnosis`, `sex`, `smoking_status`
- `primarily_metastasis`, `primary_tumor_site`, `icd_code`
- **直接提供** `cT_stage`（根据来自 pathological_data.json 的 pT 字段+病史（部分病史会直接提供）推断）
- **直接提供** `cN_stage`（根据来自 pathological_data.json 的 pN 字段+病史（部分病史会直接提供）推断）
- `medical_history`（来自 histories_english/）
- **不提供** `hpv_association_p16`（任何情况均不提供阴/阳性结果，需要模型提出要求）
- **不提供** `adjuvant_treatment_intent`（告知模型：若未给出视为no）

**口咽癌附加子任务**：
- 系统提供 cT/cN 后，先询问"下一步是否需要额外信息才能给出治疗路径？如果有则直接给出需要的，没有则回答无"//所有case均需要询问
- GT：模型必须回答"需要 p16/HPV 检测结果"，系统随即提供 p16 结果
- 模型再给出最终 NCCN 路径和治疗方案
- 此子任务独立评分，不计入总分，单独报告

**模型必须输出**：
```json
{
  "current_phase": "STAGING_AND_TREATMENT_DECISION",
  "cT_stage_confirmed": "cT3",
  "cN_stage_confirmed": "cN2b",
  "nccn_pathway_node": "HYPO-3",
  "nccn_pathway_reasoning": "肿瘤部位→T→N→节点的完整推演逻辑",
  "recommended_treatment_options": ["放疗或同步全身治疗/放疗", "手术", "诱导化疗", "临床试验"],
  "surgery_detail": "部分或全喉下咽切除术+颈清±甲状腺切除+气管旁清扫",
  "recommendation_category": "2A类",
  "next_action": "SURGERY_EXECUTION"
}
```
**商榷**：
- 手术内容是否需要单独列出（篇幅是否过长），还是和options放一起（指南格式）

**约束**：
- 非口咽癌不应要求 p16（否则扣分）
- 下咽癌/喉癌：p16 检测对这些癌种无临床意义，不要求
- 非手术首选患者（`is_non_surgical: true`）→ Step 1后直接跳到 Step 4//本数据集没有
- 数据集无帕博利珠单抗治疗，需在 Prompt 中明确告知

---

### Step 2：手术报告 → 大体病理（初步）
**阶段**：`SURGERY_GROSS_PATHOLOGY`

**输入**（step2_input.json）：
- `surgery_report`（来自reports_english/，应数据清洗删去医生辅助治疗建议）
- `medical_history`
- `icd_code`, `ops_codes`
- `number_of_positive_lymph_nodes`, `number_of_resected_lymph_nodes`

**零幻觉硬约束**：evidence 必须直接引用手术报告原文短语；完全未提及的特征填写 `"Not Av大模型lable"`，严禁医学推断

**必须输出**（未输出不给分）：
- `histologic_type` + `histotype_evidence`
- `preliminary_pT_stage`（用"至少pTX"表述）+ `pT_stage_evidence`
- `preliminary_resection_status`（R0/R1/R2/Uncert大模型n）+ `resection_evidence`
- `next_action`（指向 WSI 显微病理确认）

**pN 处理**：若报告提及转移可输出 `"pN+"`，不要求精确分期，精确分期由 Step 3 完成

---

### Step 3：WSI 切片 → 微观病理更新（多模态）
**阶段**：`WSI_MICROSCOPIC_CONFIRMATION`

**输入（商榷）**：
- `primary_5x_scalebar.jpg`（全景，含比例尺）
- `primary_20x_scalebar.jpg`（细节，含比例尺）
- `lymphnode_10x_scalebar.jpg` （含比例尺）
- Step 2 模型输出摘要

**必须输出（全部字段）**：

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
- `infiltration_depth_in_mm`（必须从比例尺测量）
- `closest_resection_margin_in_cm`（或注明无法测量）

淋巴结微观特征：
- `perinodal_invasion`（yes/no/Cannot Determine/not_applicable）
- `pN_stage`（精确分期）

---

**商榷**：
- 如何输入大像素的svs切片影像

### Step 4：最终病理汇总 → 辅助治疗决策
**阶段**：`ADJUVANT_TREATMENT_DECISION`

**输入**：Steps 1-3 的模型自身输出（累积上下文，非 GT）+promot（规范输出格式）

**两个子任务**：
- **子任务A（20%）**：汇总 Steps 1-3 全部信息，输出完整患者画像（含所有 Cannot Determine 字段）//可商榷该任务是否有必要
- **子任务B（80%）**：按顺序逐项输出辅助治疗决策

**辅助治疗决策输出顺序**：
```json
{
  "adjuvant_treatment_intent": "curative/palliative/no",
  "adjuvant_radiotherapy_y_n": "yes/no",
  "adjuvant_radiotherapy_type": "经皮放射治疗/近距离放射治疗",
  "adjuvant_systemic_therapy_y_n": "yes/no",
  "adjuvant_systemic_therapy_mode": "氟尿嘧啶+顺铂/顺铂/...",
  "adjuvant_chemoradiotherapy_y_n": "yes/no",
  "reasoning": "基于哪条不良特征+哪个NCCN节点"
}
```

**路径切换（融贯性核心测试）**：
- Step 3 更新的 pT > Step 1 直接提供的 cT（发生升期）→ Step 4 必须：
  1. 明确说明升期及原因
  2. 重新定位 NCCN 节点（如 HYPO-3 → HYPO-5）
  3. 将升期列为触发辅助治疗的依据之一
- 降期同理
- 任何缺失 → `[CASCADE_ERROR]` 惩罚
- promot中需询问是否有分期升降

---

## 6. NCCN 路径完整映射表（Step 1 GT 规则引擎，已人工修订）

根据 NCCN 2026.V1 编写，之后应考虑指南更新可能。

### 口腔癌 (Oral Cavity)
| 临床分期 | 节点 |
|----------|------|
| T1-2, N0 | OR-2 |
| T3,N0 / T1-3,N1-3 / T4a,N0-3 | OR-3 |
| T4b,N0-3 / 无法切除淋巴结 | ADV-1 |
| 不适合手术 / M1 | ADV-2 |

**OR-2**（推荐等级：无固定等级）：
- 手术（首选）；次选根治性放疗 RT
- 手术内容：原发灶切除±颈清 或 原发灶切除+SLN活检（SLN pN0→end；pN+或鉴别不成功→颈清）

**OR-3**（推荐等级：无固定等级）：
- 手术（首选）；拒绝则转ADV
- 手术内容：N0/N1/N2a-b/N3：原发灶切除+/-同侧或双侧颈清；N2c：原发灶切除+/-同侧或双侧颈清

---

### 口咽癌 p16阴性 (Oropharynx p16-)
| 临床分期 | 节点 |
|----------|------|
| T1-2, N0-1 | ORPH-2 |
| T3-4a, N0-1 | ORPH-3 |
| T1-4a, N2-3 | ORPH-4 |
| T4b,N0-3 / 无法切除 | ADV-1 |
| M1 | ADV-2 |

注：NCCN 2026 中 ORPH-3/4 决策树相同，但手术细节因病情不同有区别

**ORPH-2**（手术/RT标准；放化疗2B类）：
- 手术；根治性放疗RT；同步全身治疗/放疗（仅T1-2,N1，2B类）；临床试验
- 手术内容：原发灶切除+同侧或双侧颈清

**ORPH-3/4**（放化疗/手术标准；诱导3类）：
- 同步全身治疗/放疗；手术；诱导化疗（3类）随后进行放疗或全身治疗/放疗；临床试验
- 手术内容：原发灶切除+同侧或双侧颈清

---

### 口咽癌 p16阳性 (Oropharynx p16+/HPV+)
| 临床分期 | 节点 |
|----------|------|
| M1 | ADV-2 |
| T1-2, N0 | ORPHPV-1 |
| T0-2, N1（单节点≤3cm） | ORPHPV-2 |
| T0-2,N1（单节点>3cm或同侧≥2个≤6cm）/ T1-2,N2 / T3,N0-2 | ORPHPV-3 |
| T0-3,N3 / T4,N0-3 | ORPHPV-4 |

**ORPHPV-1**（无固定等级）：手术；根治性放疗RT；临床试验
- 手术内容：原发灶切除+同侧或双侧选择性颈清

**ORPHPV-2**（手术/RT标准；放化疗2B类）：手术；根治性放疗RT；同步放化疗（2B类）；临床试验
- 手术内容：原发灶切除+同侧或双侧颈清

**ORPHPV-3**（放化疗/手术标准；诱导3类）：同步全身治疗/放疗；手术；诱导化疗（3类）随后进行放疗或全身治疗/放疗；临床试验
- 手术内容：原发灶切除+同侧或双侧颈清

**ORPHPV-4**（放化疗首选）：同步全身治疗/放疗（首选）；手术；诱导化疗（3类），随后进行放疗或同步全身治疗/放疗；临床试验
- 手术内容：原发灶切除+同侧或双侧颈清

---

### 下咽癌 (Hypopharynx)
| 临床分期 | 节点 |
|----------|------|
| 适合喉保留手术（大多数T1,N0；部分T2,N0） | HYPO-2 |
| 需咽切除+全喉切除：T1-3, N0-3 | HYPO-3 |
| 需咽切除+全喉切除：T4a, N0-3 | HYPO-5 |
| T4b,N0-3 / 无法切除/不适合手术 | ADV-1 |
| M1 | ADV-2 |

注：p16 检测对下咽癌无临床意义，不要求

**HYPO-2**（无固定等级）：根治性放疗RT；手术；临床试验
- 手术内容：部分喉下咽切除（开放/内镜）+同侧或双侧颈清±甲状腺切除+气管前和同侧气管旁淋巴结清扫

**HYPO-3**（2A类）：放疗或同步全身治疗/放疗；手术；诱导化疗；临床试验
- 手术内容：部分或全喉下咽切除+同侧或双侧颈清±甲状腺切除+气管前和同侧气管旁淋巴结清扫

**HYPO-5**（手术标准；非手术3类）：手术；诱导化疗（3类）；同步全身治疗/放疗（3类）；临床试验
- 手术内容：全喉下咽切除+同侧或双侧颈清±甲状腺切除+气管前和同侧气管旁淋巴结清扫

---

### 声门喉癌 (Glottic Larynx)
| 临床分期 | 节点 |
|----------|------|
| 原位癌 / 适合保留喉功能（T1-2,N0；选定T3,N0） | GLOT-2 |
| T3需全喉切除，N0-1 | GLOT-3 |
| T3需全喉切除，N2-3 | GLOT-4 |
| T4a | GLOT-6 |
| T4b,N0-3 / 无法切除/不适合手术 | ADV-1 |
| M1 | ADV-2 |

**GLOT-2**（无固定等级）：
- 原位癌：内镜切除术（首选）或放疗
- 适合保留喉功能（T1-T2,N0或选定T3,N0）：放疗；手术
- 手术内容：根据指征行部分喉切除术 / 内镜或开放性切除术+颈清

**GLOT-3**（无固定等级）：同步全身治疗/放疗（不适合全身治疗时单纯RT）；手术；诱导化疗；临床试验
- 手术内容：同侧或双侧颈清+考虑甲状腺切除以清除中央区淋巴结

**GLOT-4**（无固定等级）：同步全身治疗/放疗；手术；诱导化疗；临床试验
- 手术内容：根据指征行喉切除联合甲状腺切除+同侧或双侧颈清+气管前和同侧气管旁淋巴结清扫

**GLOT-6**（手术标准）：T4a,N0-3手术（首选）；拒绝或入选T4a患者则考虑放化疗/临床试验/诱导化疗
- 手术内容：同侧或双侧颈淋巴结清扫+为清除中央区淋巴结而进行的甲状腺切除术（特别是甲状腺存在喉外侵犯及明显声门下扩展时）

---

### 声门上喉癌 (Supraglottic Larynx)
| 临床分期 | 节点 |
|----------|------|
| 适合保留喉功能（大多T1-2,N0；部分T3） | SUPRA-2 |
| 需全喉切除，T3,N0 | SUPRA-3 |
| T4a, N0 | SUPRA-8 |
| 淋巴结阳性 → SUPRA-4分流 | |
| → 适合保留喉（T1-2,N+；部分T3,N1） | SUPRA-5 |
| → 需全喉切除（大多T3,N1-3） | SUPRA-6 |
| → T4a, N1-3 | SUPRA-8 |
| T4b,N0-3 / 无法切除/不适合手术 | ADV-1 |
| M1 | ADV-2 |

**SUPRA-2**（无固定等级）：手术（适合保留喉功能者：大多数T1-2,N0；选定T3期）；RT
- 手术内容：内镜或开放部分喉切除+颈清

**SUPRA-3**（无固定等级）：同步全身治疗/放疗或单纯RT（患者身体不适合时）；手术；诱导化疗；临床试验
- 手术内容：喉切除术+甲状腺切除术+同侧、中央组或双侧颈清扫术

**SUPRA-5**（无固定等级）：同步放化疗或RT或根治性放疗（肿瘤负荷较低T1-2,N1或不适合全身治疗）；手术；诱导化疗；临床试验
- 手术内容：内镜或开放部分喉切除+颈清

**SUPRA-6**（无固定等级）：同步全身治疗/放疗；手术；诱导化疗；临床试验
- 手术内容：喉切除+同侧甲状腺切除+颈清

**SUPRA-8**（无固定等级）：手术（T4a,N0-N3首选）；拒绝则考虑放化疗/临床试验/诱导化疗
- 手术内容：内镜或开放部分喉切除+颈部淋巴结清扫术

---

## 7. 辅助治疗完整决策树（Step 4 GT，已修订）

### 不良病理特征分级

| 级别 | 特征 | 结果 | 推荐等级 |
|------|------|------|----------|
| 🔴 高危 | 淋巴结外侵犯(perinodal=yes) 和/或 切缘阳性(R1/margin<0.1cm) | 系统治疗+放疗（同步放化疗） | **1类** |
| 🟡 其他风险 | pT3/T4、pN2/N3、多淋巴结阳性、LVI+、Pn+、单淋巴结阳性无其他特征 | 放疗 或 考虑系统治疗/放疗 | 2A类 |
| 🟢 无不良特征 | pN0 或 pN1且无其他风险 | 观察(end) 或 考虑RT | — |

注：数据集不存在帕博利珠单抗治疗，Prompt中须告知模型。
辅助系统治疗模式选项：氟尿嘧啶+顺铂 / 顺铂+多西他赛 / 氟尿嘧啶+卡铂 / 顺铂 / 西妥昔单抗

### 各类型辅助治疗决策（完整版，已修订）

**【口腔癌 OR-2 术后】**
- 无阳性淋巴结，无不良特征 → end（观察）
- 1个阳性淋巴结，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗（1类）
- 切缘阳性（无ENE）→ 再次切除（如果可行）；切缘阴性后考虑 RT；or 考虑全身治疗/放疗
- 其他风险特征 → RT；考虑全身治疗/放疗

**【口腔癌 OR-3 术后】**
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗（1类）
- 切缘阳性（无ENE）→ 系统治疗/放疗（1类）or 再次切除（若可行，切缘阴性后考虑RT）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【口咽癌 p16- ORPH-2 术后】**
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯（±切缘阳性）→ 系统治疗/放疗
- 切缘阳性 → 再次切除（如果可行）or 放疗 or 考虑系统治疗/放疗
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【口咽癌 p16- ORPH-3/4 术后】**
- 无不良特征 → RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【口咽癌 p16+ ORPHPV-1 术后】**
- 无不良特征 → end（观察）
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（2B类）
- 切缘阳性 → 再次切除（如果可行）or 考虑系统治疗/放疗 or 放疗
- 其他风险特征 → 放疗 or 系统治疗/放疗（2B类）

**【口咽癌 p16+ ORPHPV-2 术后】**
- 无不良特征 → end（观察）
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（2B类）
- 切缘阳性 → 再次切除（如果可行）or 考虑系统治疗/放疗 or 放疗（2B类）
- 其他风险特征 → 放疗 or 系统治疗/放疗（2B类）

**【口咽癌 p16+ ORPHPV-3 术后】**
- 无不良特征 → end（观察）
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗
- 其他风险特征 → 放疗 or 系统治疗/放疗

**【口咽癌 p16+ ORPHPV-4 术后】**
- 无不良特征 → end（观察）
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【下咽癌 HYPO-2 术后】**
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 切缘阳性 → 再次切除（如果可行）or 考虑系统治疗/放疗（仅适用于T2期）or 放疗
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【下咽癌 HYPO-3 术后】**
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【下咽癌 HYPO-5 术后（pT4a升期后切入）】**
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【声门喉癌 GLOT-2 术后】**
- 原位癌 → end
- T1-T2,N0或部分T3,N0，无其他不良特征 → 观察
- N1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 → 系统治疗/放疗（1类）
- 切缘阳性 → 再次切除（如果可行）or 放疗
- 其他风险特征 → 放疗

**【声门喉癌 GLOT-3 术后】**
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【声门喉癌 GLOT-4 术后】**
- 无不良特征 → end
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【声门喉癌 GLOT-6 术后】**
- 无不良特征 → end
- N1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他风险特征 → 放疗 or 考虑系统治疗/放疗

**【声门上喉癌 SUPRA-2 术后】**
- 淋巴结阴性（T1-2,N0）→ end
- 单个淋巴结阳性，无其他不良特征 → 考虑 RT
- 淋巴结阳性；切缘阳性 → 再次切除（仅高度筛选可行者）or RT or 考虑系统治疗/放疗
- 🔴 淋巴结外侵犯 → 系统治疗/放疗（1类）or 放疗（2B类，特定筛选患者）
- 淋巴结阳性+其他不良特征 → 放疗 or 考虑系统治疗/放疗
- 淋巴结阴性（T3-T4a,N0）→ 进入对应 SUPRA-3/8 辅助决策

**【声门上喉癌 SUPRA-3 术后】**
- pN0，无不良特征 → end
- pN1，无其他不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他不良特征 → 放疗 or 考虑系统治疗/放疗

**【声门上喉癌 SUPRA-5 术后】**
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他不良特征 → 放疗 or 考虑系统治疗/放疗

**【声门上喉癌 SUPRA-6 术后】**
- 无不良特征 → 考虑 RT
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他特征 → 放疗 or 考虑系统治疗/放疗

**【声门上喉癌 SUPRA-8 术后】**
- 🔴 淋巴结外侵犯 和/或 切缘阳性 → 系统治疗/放疗（1类）
- 其他不良特征 → 放疗 or 考虑系统治疗/放疗

---

## 8. 评分体系（需商榷具体细则）

### 总分结构
```
步骤权重：Step1=25%, Step2=20%, Step3=30%, Step4=25%
每步内部：60%×结构化字段 + 40%×LLM-Judge
Step4：20%×汇总子任务 + 80%×决策子任务
Step1口咽癌附加子任务：独立评分，不计入总分，单独报告
Case总分：Σ(权重×步骤分) × safety_multiplier
safety_multiplier：max(0.5, 1.0 - critical_error_count × 0.2)
双轨：AR Score（级联传播）和 IND Score（每步注入GT）分别报告
```

### 结构化字段评分

| 字段类型 | 满分=1.0 | 部分分=0.5 | 零分=0 |
|----------|---------|-----------|--------|
| 分期（cT/pT/cN/pN） | 精确匹配 | 相邻分期 | 跨越两级+ |
| NCCN节点 | 精确匹配 | 同部位相邻节点 | 不同部位节点 |
| 推荐等级 | 精确 | 1类↔2A（方向对） | 1类↔3类 |
| 无推荐等级节点（OR-2等） | 不评分此字段 | — | — |
| 二元yes/no（🔴字段） | 精确；Cannot Determine+合理理由=满分 | — | 反向输出 |
| R0/R1/R2 | 精确 | R0↔R1=0.3 | R0↔R2 |
| 数值（DOI,margin） | 误差≤0.05cm/1mm | 误差0.05~0.1cm/1~3mm | 超出范围 |
| 辅助治疗yes/no | 精确 | — | 反向（触发CSS惩罚） |

### 临床安全惩罚（CSS，独立报告）

| 错误 | Flag | 惩罚 |
|------|------|------|
| 漏报perinodal | [PERINODAL_MISS] | Step3×0.3；CSS-0.30 |
| 漏开放化疗 | [CHEMORT_MISSED] | Step4决策×0.2；CSS-0.30 |
| 阶段时序混淆 | [TEMPORAL_VIOLATION] | 违规字段清零 |
| 幻觉引用 | [HALLUCINATION_FLAG] | evidence字段清零；CSS-0.10 |
| 无WSI输出yes/no | [LN_WSI_HALLUC] | 该字段×0.2；CSS-0.10 |
| 升期后路径未切换 | [CASCADE_ERROR] | 额外-0.15 |

```
CSS = 1.0 - (高危漏报×0.30) - (幻觉次数×0.10)
```

---

## 9. 特殊案例处理

| 情况 | meta标记 | 处理 |
|------|---------|------|
| 非手术首选（本数据集没有） | `is_non_surgical: true` | Step1后直接跳Step4，Step2/3标记NA |
| 声门/声门上区分 | `icd_codes_larynx_subtype` | 大模型按ICD归类，专家全量检查 |
| 病史透露cTcN | `cont大模型ns_cTcN_leak: true` | 删除泄露内容，作为信息更新能力测试 |
| 患者拒绝辅助 | `patient_declined: true` | 按`guideline_recommends`字段评分 |
| 数据矛盾患者 | `data_contradiction: true` | 专家修正后提供正确GT |
| 531号口咽癌p16未测 | 建议剔除 | 不纳入评测集 |

pNX(20%数据缺失)
| 场景 | cN（术前| pN（术后病理）|说明|
|------|---------|------|------|
|正常案例|cN0/cN1/cN2/cN3|pN0/pN1/pN2/pN3|正常处理|
|未做淋巴颈清|cN0|pNX|术前评估阴性，未做颈清，无病理结果|
|颈清但未评估|cN+|pNX|数据缺失，建议排除或另设难度层次问题检查是否存在幻觉|

---

## 10.需商榷部分总结

- **Step 1（初始治疗决策）的输出格式**：关于nccn指南给出的手术内容，商榷是需要将其单独列出（需考虑篇幅是否过长），还是应当和治疗选项（options）放在一起以符合指南的格式。
- **Step 3（WSI 切片更新）的输入内容**：商榷该步骤的具体输入物，包括带比例尺的原发灶全景图（5x）和细节图（20x）、带比例尺的淋巴结图（10x），以及 Step 2 的模型输出摘要。
- **Step 3（WSI 切片更新）的影像格式处理**：商榷如何实际向模型输入大像素的 svs 切片影像。
- **Step 4（辅助治疗决策）的子任务设置**：商榷“子任务 A”（要求汇总 Steps 1-3 的全部信息，输出包含所有 Cannot Determine 字段的完整患者画像）是否具有必要性。
- **评分体系**：整个第 8 节“评分体系”的具体评判细则尚未完全敲定，需要进一步商榷。
