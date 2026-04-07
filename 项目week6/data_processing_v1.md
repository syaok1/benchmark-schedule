---

# HNC-Benchmark 项目全景梳理

## 一、两文件冲突对照（以 context-v7 为准）

| 冲突点 | summary-v2 内容 | context-v7（正确）|
|--------|----------------|-------------------|
| **`staging_upgrade_occurred` 字段名** | `staging_upgrade_occurred` | ⚠️ **`staging_change_occurred`**（v7 B级表格） |
| **pNX 处理方式** | 按4种情况分类（pN0满分/pN+无WSI扣分/pNX未颈清/pNX记录缺失） | ⚠️ **简化为两种情况**（pN0/pN+但不可得），并新增建议："建议根据情况归类为pN0或划分为数据部分缺失的高难度题" |
| **p16子任务触发机制描述** | 先系统提供cT/cN，口咽癌患者再声明"需要p16" | ⚠️ **v7更精确**：所有case均先询问"是否需要额外信息"，GT是口咽癌必须回答"需要p16"，非口咽癌答"无" |
| **Step 2 GT字段** | 仅提 pT/R/histotype | ⚠️ **v7新增 pN_stage** 作为 Step 2 必须输出字段 |
| **`preliminary_pT_stage` B级审核重点** | "pT3 vs pT4a 边界" | ⚠️ **v7改为**"从手术报告规范提取" |
| **文件引用** | 治疗方案引用"context-v5.md" | ⚠️ **v7已内置完整映射表**，不再引用外部文件 |
| **Step 4 输出格式** | 未详述 | ⚠️ **v7新增**：`adjuvant_radiotherapy_type`/`adjuvant_systemic_therapy_mode` 等字段，并说明prompt需询问GT中记录的治疗是否正确 |
| **Case 文件结构** | step2_gt 仅含"初步pT/R/histotype + Evidence Anchors" | ⚠️ **v7**：step2_gt 含"初步pT/**pN**/R GT + evidence anchors" |

---

## 二、理想情况下各步骤的完整 Ground Truth / INPUT / OUTPUT

### Step 1：`STAGING_AND_TREATMENT_DECISION`

**INPUT（step1_input.json）**

| 字段 | 来源 | 说明 |
|------|------|------|
| `year_of_initial_diagnosis` | clinical_data.json | — |
| `age_at_initial_diagnosis` | clinical_data.json | — |
| `sex` | clinical_data.json | — |
| `smoking_status` | clinical_data.json | — |
| `primarily_metastasis` | clinical_data.json | — |
| `primary_tumor_site` | clinical_data.json | — |
| `cT_stage` | 由 pT + 病史**推断**后直接给出 | 不做影像推理，直接提供 |
| `cN_stage` | 由 pN + 病史**推断**后直接给出 | 不做影像推理，直接提供 |
| `Diagnosis / ICD codes` | clinical_data.json | — |
| `medical_history` | histories_english/ | — |
| ~~`hpv_association_p16`~~ | **不提供** | 所有case均不给，需模型主动要求 |
| ~~`adjuvant_treatment_intent`~~ | **不提供** | prompt告知未给出视为no |

**GROUND TRUTH（step1_gt.json）**

| 字段 | 来源/标注方式 | 标注级别 |
|------|-------------|---------|
| `nccn_pathway_node` | 规则引擎（tumor_site+cT+cN+p16）→ 专家抽样验证20% | A级 |
| `nccn_pathway_reasoning` | LLM-Judge评分（自由文本） | — |
| `recommended_treatment_options` | NCCN映射表（见第6节完整版） | A级 |
| `surgery_detail` | NCCN映射表 | A级 |
| `recommendation_category` | NCCN节点固有属性（部分节点无固定等级） | A级 |
| `next_action` | 固定（→SURGERY_EXECUTION） | C级自动 |
| **口咽癌附加子任务GT**：模型必须主动要求p16结果 | 规则判断（口咽癌=必须要求） | B级 |

**模型必须输出（OUTPUT样例）**
```json
{
  "current_phase": "STAGING_AND_TREATMENT_DECISION",
  "nccn_pathway_node": "HYPO-3",
  "nccn_pathway_reasoning": "肿瘤部位→T→N→节点的完整推演逻辑",
  "recommended_treatment_options": ["放疗或同步全身治疗/放疗", "手术", "诱导化疗", "临床试验"],
  "surgery_detail": "部分或全喉下咽切除术+颈清±甲状腺切除+气管旁清扫",
  "recommendation_category": "2A类",
  "next_action": "SURGERY_EXECUTION"
}
```

---

### Step 2：`SURGERY_GROSS_PATHOLOGY`

**INPUT（step2_input.json）**

| 字段 | 来源 | 预处理要求 |
|------|------|----------|
| `surgery_report` | reports_english/ | **必须删去**：①医生辅助治疗建议段落；②透露cT/cN的内容（标记leak） |
| `Tests and treatments / OPS codes` | clinical_data.json | — |
| `number_of_positive_lymph_nodes` | pathological_data.json | — |
| `number_of_resected_lymph_nodes` | pathological_data.json | — |
| `Results of laboratory tests` | blood_data.json | — |
| `first_treatment_intent` | 固定值 `"curative"` | — |
| `first_treatment_modality` | 固定值 `"local surgery"` | — |

**GROUND TRUTH（step2_gt.json）**

| 字段 | 来源/标注方式 | 标注级别 |
|------|-------------|---------|
| `histologic_type` | pathological_data.json | C级自动 |
| `histotype_evidence` | 手术报告原文短语 | **A级（LLM辅助提取，专家确认）** |
| `pT_stage`（初步，"至少pTX"表述） | 规则从手术报告提取 | B级 |
| `pT_stage_evidence` | 手术报告原文短语 | **A级（LLM辅助提取，专家确认）** |
| `pN_stage`（初步，仅判断pN+/pN-） | 手术报告关键词 | B级 |
| `pN_stage_evidence` | 手术报告原文短语 | **A级（LLM辅助提取，专家确认）** |
| `resection_status`（R0/R1/R2/Uncertain） | pathological_data.json + 报告 | B级 |
| `resection_evidence` | 手术报告原文短语 | **A级（LLM辅助提取，专家确认）** |
| `next_action` | 固定（→WSI确认） | C级自动 |

**硬约束**：evidence字段必须直接引用报告原文；完全未提及的特征必须输出 `"Not Available"`，严禁医学推断。

---

### Step 3：`WSI_MICROSCOPIC_CONFIRMATION`

**INPUT（step3_images/，技术方案商榷中）**

| 文件 | 来源 | 说明 |
|------|------|------|
| `primary_5x_scalebar.jpg` | WSI_PrimaryTumor/*.svs + GeoJSON | ROI提取，含比例尺 |
| `primary_20x_scalebar.jpg` | 同上 | 细节图 |
| `lymphnode_10x_scalebar.jpg` | WSI_LymphNode/*.svs | 缺失时打标记 |
| Step 2 模型输出摘要 | 上一步输出 | AR模式下为模型输出，IND模式下为GT |

**GROUND TRUTH（step3_gt.json）**

| 字段 | 来源 | 标注级别 |
|------|------|---------|
| `pT_stage`（更新） | pathological_data.json | C级自动 |
| `pN_stage`（精确分期） | pathological_data.json | C级自动 |
| `resection_status` | pathological_data.json | C级自动 |
| `histologic_type` | pathological_data.json | C级自动 |
| `perinodal_invasion` | pathological_data.json（yes/no） | **C级自动（🔴高危，评分加权）** |
| `lymphovascular_invasion_L` | pathological_data.json | C级自动 |
| `vascular_invasion_V` | pathological_data.json | C级自动 |
| `perineural_invasion_Pn` | pathological_data.json | C级自动 |
| `grading_hpv`（G1/G2/G3） | pathological_data.json | C级自动 |
| `resection_status_carcinoma_in_situ` | pathological_data.json | C级自动 |
| `carcinoma_in_situ` | pathological_data.json | C级自动 |
| `closest_resection_margin_in_cm` | pathological_data.json | C级自动 |
| `infiltration_depth_in_mm` | pathological_data.json（需从比例尺测量） | C级自动 |

**淋巴结WSI缺失GT对照**

| 情况 | 模型正确输出 | 评分 |
|------|------------|------|
| pN0（无转移） | `"not_applicable"` | 满分 |
| pN+ 但WSI不可得 | `"Cannot Determine"` | 满分；输出yes/no扣分+CSS |

---

### Step 4：`ADJUVANT_TREATMENT_DECISION`

**INPUT**：Prompt（规范输出格式） + Steps 1-3 全部输出汇总

**GROUND TRUTH（step4_gt.json）**

| 字段 | 来源/标注方式 | 标注级别 |
|------|-------------|---------|
| `adjuvant_treatment_intent` | pathological_data.json + 规则引擎 | B级 |
| `adjuvant_radiotherapy_y_n` | pathological_data.json | C级自动 |
| `adjuvant_radiotherapy_type` | pathological_data.json | C级自动（prompt再次确认） |
| `adjuvant_systemic_therapy_y_n` | pathological_data.json | C级自动 |
| `adjuvant_systemic_therapy_mode` | pathological_data.json | C级自动（prompt再次确认） |
| `adjuvant_chemoradiotherapy_y_n` | 规则引擎（高危特征判断） | B级 |
| `reasoning` | 基于不良特征+NCCN节点 | LLM-Judge |
| **指南偏差**：`patient_declined: true` + `guideline_recommends` | **A级专家确认** | A级 |
| **数据矛盾修正** | **A级专家确认** | A级 |
| `adjuvant_decision_trigger` | 从病理特征自动生成 | B级 |
| `staging_change_occurred` | pT vs cT 自动比较 | B级 |

**路径切换GT**：若pT > cT（升期），GT要求模型必须：① 说明升期原因 ② 重新定位NCCN节点 ③ 将升期列为辅助治疗触发依据。

---

## 三、数据处理方法标注方案

> 下表为**建议方案**，请您增删修改

### A级：必须专家介入（LLM可生成初稿）

| # | 任务 | 处理方法 | LLM做什么 | 专家做什么 | 脚本 |
|---|------|---------|----------|----------|------|
| A1 | NCCN路径节点（Step 1 GT） | 规则引擎自动定位 | 生成候选节点 + reasoning草稿 | 抽样验证20%，重点T3/T4a边界、声门上N+分流 | `nccn_router.py` |
| A2 | Evidence Anchor提取（Step 2 GT） | LLM从手术报告定位原文短语 | 提取pT/pN/R/histotype对应的关键原文短语候选（无则标NULL） | 全量确认/修改 | `extract_evidence_anchors.py` |
| A3 | 指南偏差-类型A（患者拒绝辅助） | LLM筛选"应治疗但未治疗"案例 | 标记候选案例，生成`guideline_recommends`草稿 | 最终确认，标注`patient_declined: true` | `detect_data_issues.py` |
| A4 | 指南偏差-类型B（数据矛盾） | LLM检测内部矛盾字段 | 输出矛盾对（如"系统疗法:no"但"模式:顺铂"） | 根据NCCN指南修正，给出正确GT | `detect_data_issues.py` |

---

### B级：自动为主 + 专家抽样10-20%

| # | 字段 | 处理方法 | LLM是否参与 | 审核重点 | 脚本 |
|---|------|---------|-----------|---------|------|
| B1 | `preliminary_pT_stage`（Step 2 GT） | 规则从手术报告关键词提取 | 否（规则） | 边界案例，从手术报告规范提取 | `parse_hancock.py` |
| B2 | `staging_change_occurred` | pT vs cT 自动数值比较 | 否（规则） | pT < cT 异常降期案例 | `case_builder.py` |
| B3 | `high_risk_gross_signals` | LLM关键词提取 | **是** | 与perinodal字段对应关系 | `extract_evidence_anchors.py` |
| B4 | `adjuvant_decision_trigger` | 规则引擎（pT/pN/R/LVI/Pn → 不良特征判断） | 否（规则） | 多特征并存优先级（🔴 vs 🟡） | `adjuvant_router.py` |
| B5 | ICD codes 声门/声门上分类 | LLM按ICD codes归类为glottic/supraglottic | **是（全量处理）** | **专家全量检查**（影响NCCN路径） | `classify_larynx_icd.py` |
| B6 | 口咽癌p16状态（`hpv_association_p16`） | 从pathological_data.json提取 | LLM辅助异常值判断 | 检查531号剔除，hpv_association_p16字段核实 | `parse_hancock.py` |
| B7 | `cT_stage` / `cN_stage` 推断 | 由pT/pN + 病史规则推断 | LLM兜底（隐含分期描述） | 核查leak标注一致性 | `detect_cTcN_leak.py` |

---

### C级：完全自动（无需专家）

| # | 字段 | 处理方法 | 脚本 |
|---|------|---------|------|
| C1 | 所有yes/no二元字段（perinodal/LVI/Pn等） | 直接从pathological_data.json读取 | `parse_hancock.py` |
| C2 | `current_phase` | 按步骤硬编码 | `case_builder.py` |
| C3 | 淋巴结WSI缺失标记 | 文件存在性检查 | `record_ln_wsi_status.py` |
| C4 | `p16_required` | 口咽癌=true，其他=false | `parse_hancock.py` |
| C5 | 难度分级（Simple/Medium/Hard） | 按Stage I-II/III/IV自动分配 | `case_builder.py` |
| C6 | `is_non_surgical` | 从clinical_data.json判断（本数据集均为false） | `parse_hancock.py` |
| C7 | 辅助治疗建议删除标记 | 规则匹配（LLM兜底） | `clean_surgery_reports.py` |
| C8 | `contains_cTcN_leak` | 规则匹配+LLM兜底 | `detect_cTcN_leak.py` |
| C9 | pNX分类 | 按cN+手术记录规则分类，pNX数据缺失→高难度题/排除 | `detect_data_issues.py` |

---

### 特别说明：三个需要LLM的清洗模块

| 模块 | LLM任务 | 人工校验 |
|------|---------|---------|
| `extract_evidence_anchors.py` | 从手术报告定位支持pT/pN/R/histotype的原文短语 | A级专家全量确认 |
| `clean_surgery_reports.py` | 识别并删除医生辅助治疗建议段落 | 抽样10%核查 |
| `detect_cTcN_leak.py` | 规则匹配未命中时，识别隐含临床分期描述 | 抽样核查 |

---

以上方案审阅，需对各条目进行增删修改（特别是B5的ICD专家全量检查是否资源允许、A2的Evidence Anchor是否全量专家还是抽样）。
