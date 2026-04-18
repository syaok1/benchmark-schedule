# HNC-Benchmark 项目 Context（新对话用）

**版本：2026-04 | 状态：设计完成，进入工程搭建阶段**

---

## 一、项目概述

**目标**：构建评测多模态大语言模型（MLLM）在头颈癌临床推理能力的 Benchmark，公开网站（支持本地部署）。

**数据**：Hancock 真实患者数据集，763 例，德国 Erlangen 单中心，2005–2019。
**指南**：NCCN Head and Neck Cancers 2026.V1。
**癌种**：口腔癌 / 口咽癌(p16+) / 口咽癌(p16−) / 下咽癌 / 声门喉癌 / 声门上喉癌。

---

## 二、四步推理链（核心设计，已锁定）

```
Step1: STAGING_AND_TREATMENT_DECISION
  输入：临床数据（cT/cN、ICD、病史）
  输出：cancer_subtype_resolved / nccn_pathway_node / recommended_treatment_options / next_action
  特殊：口咽癌需两轮（Turn1 请求 p16 → 注入 p16 结果 → Turn2 完整输出）

Step2: SURGERY_GROSS_PATHOLOGY
  输入：已清洗手术报告（含 [REDACTED_*] 标记）
  输出：histologic_type / preliminary_pT_stage / preliminary_resection_status
        / preliminary_pN_stage / *_evidence（4 个 verbatim 引用）

Step3: WSI_MICROSCOPIC_CONFIRMATION
  输入：多倍率 WSI JPEG（5x/20x 原发灶 + 10x 淋巴结，含比例尺）
  输出：updated_pT/pN/R / 微观侵袭特征（PNI/LVI/ENE）/ grading / 测量值

Step4: ADJUVANT_TREATMENT_DECISION（单轮）
  输入：Step1–3 汇总 + 淋巴结计数
  输出：Adverse_pathologic_features / NCCN_recommended_adjuvant_treatment
        / adjuvant_* y/n / staging_change / change_triggers_nccn_switch
```
### 详细INPUTOUTPUT：
- 严格照搬HNC_Benchmark_Complete_Integrated_v1.md内容，不再赘述。
---

## 三、双榜评分体系（已锁定，不修改）

### 榜单 1：Full-Chain 联通榜
链式截止，得分 ∈ {0.00, 0.25, 0.50, 0.75, 1.00}

| 截止步骤 | 得分 |
|---|---|
| Step1 Fail | 0.00 |
| Step2 Fail | 0.25 |
| Step3 Fail | 0.50 |
| Step4 Fail | 0.75 |
| 全部 Pass | 1.00 |

### 榜单 2：Per-Step 独立榜
每步独立注入 GT prior_context，Si ∈ {0, 1}，`per_step_total = (S1+S2+S3+S4)/4`

### 两榜共用 Pass/Fail 判定规则

| Step | 判定项 | 方式 | 关键规则 |
|---|---|---|---|
| Step1 | #1 nccn_pathway_node | SCRIPT | ==GT 且不跨癌种家族 |
| | #2 next_action | SCRIPT | ==GT |
| | #3 Cat1 治疗选项不漏 | **LLM** | 推荐等级一定要正确，初步治疗方案应全面覆盖指南内容 |
| | #4 p16 请求意图 "need_more_info": "p16_status" | **LLM** | 口咽癌语义意图正确；非口咽癌必须 null |
| Step2 | #1 histologic_type | SCRIPT | ==GT（SCC↔adeno 直接 Fail）|
| | #2 preliminary_pT_stage | SCRIPT | ==GT 严格 |
| | #3 preliminary_resection_status | SCRIPT | ==GT（R0↔R1/R2 直接 Fail）|
| | #4 preliminary_pN_stage | SCRIPT | ==GT（要求在pN0 \| pN+ \| Not Available中选择;出现pN0↔pN+ 直接 Fail）|
| | #5 *_evidence 4 个 anchor | **LLM** | 4/4 全有效；含 [REDACTED_*] 自动 invalid |
| Step3 | #1 updated_pT_stage | SCRIPT | ==GT 严格 |
| | #2 updated_pN_stage | SCRIPT | ==GT 严格；WSI缺失豁免 |
| | #3 updated_resection_status | SCRIPT | ==GT（R0↔R1/R2 直接 Fail）|
| | #4 updated_histologic_type| SCRIPT | ==GT 严格；WSI缺失豁免 |
| | #5 perineural_invasion_Pn | SCRIPT | ==GT 严格 |
| | #6 lymphovascular_invasion_Lv | SCRIPT | ==GT 严格 |
| | #7 vascular_invasion_V | SCRIPT | ==GT 严格 |
| | #8 grading_hpv | SCRIPT | ==GT 严格 |
| | #9 carcinoma_in_situ | SCRIPT | ==GT 严格 |
| | #10 resection_status_carcinoma_in_situ | SCRIPT | ==GT 严格 |
| | #11 infiltration_depth_in_mm | SCRIPT | ==GT ±1mm |awd
| | #12 closest_resection_margin_in_cm | SCRIPT | ==GT ±0.1cm |
| | #13 perinodal_invasion | SCRIPT | ==GT 严格；WSI缺失豁免 |
| | #14 next_action | SCRIPT | ==GT |
| Step4 | #1 NCCN  辅助治疗 | **HYBRID** | SCRIPT预判 Cat1 全覆盖 + LLM 判额外选项合理性 |
| | #2 不良特征 "Adverse_pathologic_features_present","Adverse_pathologic_features"asd| **LLM** | GT 关键特征（ENE/阳性切缘）不漏；无幻觉 |
| | #3 STAGING CHANGE ASSESSMENT | SCRIPT | ==GT |

### LLM-Judge（4 类任务）
- **Prompt-A**：p16 请求意图（Step1 口咽癌）
- **Prompt-B**：Evidence anchor 有效性（Step2，4/4 才 pass）
- **Prompt-C**：NCCN 额外治疗选项合理性（Step4 HYBRID 兜底）
- **Prompt-D**：不良特征合理性（Step4）

### Leaderboard（16 个榜单）
主榜 2（FC + PS）+ Step独立榜 4 + 癌种子榜 10（5 癌种 × FC/PS 各一）

---

## 四、数据集关键事实

| 模态 | 数量 | 可用率 | 说明 |
|---|---|---|---|
| 原发灶 WSI | 709 张 | 92% | SVS，82.44×，0.1213µm/px |
| 淋巴结 WSI | 396 张 | **52%** | 两种扫描仪 |
| GeoJSON 标注 | ~701 个 | ~100% | QuPath 标注，用于 ROI 提取 |
| 手术报告（英译）| >98% | >98% | DeepL 翻译，reports_english/ |
| 病理结构化数据 | 763 例 | 100% | pathological_data.json |
| 临床结构化数据 | 763 例 | 100% | clinical_data.json |

**关键限制**：
- 淋巴结 WSI 缺失 ~367 例 → Step3 ENE/pN 字段豁免为 `Cannot Determine`
- Hancock 声门/声门上未区分 → 需按 ICD code 区分
- 约 20% 案例 pN 记录为 NX → 归类为 pN0 或标记 `partial_info_missing`
- 口咽癌 p16 缺失案例（如 531 号）→ 排除

---

## 五、Case 数据结构

```
Case_XXXX/
├── meta.json              # 案例元数据、排除标记、clinical_scenario
├── step1_input.json       # 临床分期输入（cT/cN，不含 p16 结果）
├── step1_gt.json          # NCCN 节点 GT + 治疗选项 GT
├── step2_input.json       # 已 REDACT 的手术报告 + 结构化字段
├── step2_gt.json          # histotype/pT/R/pN + evidence anchors
├── step3_input.json       # WSI 图像路径（多倍率 JPEG）+ prior_context
├── step3_gt.json          # 微观病理 GT（来自 pathological_data.json）
├── step4_input.json       # 汇总输入 + adjuvant_treatment_intent
└── step4_gt.json          # 辅助治疗决策 GT

```

---

## 六、评分框架工程架构（Round 6）

```
hnc_benchmark/
├── runner/
│   ├── pipeline.py           # FullChainPipeline（串行）+ PerStepPipeline（并行）
│   ├── scoring_context.py    # ScoringContext dataclass
│   └── aggregator.py         # 16 个 leaderboard + 5 个独立统计指标
├── evaluators/
│   ├── step1_evaluator.py    # SCRIPT×4 + LLM×1
│   ├── step2_evaluator.py    # SCRIPT×4 + LLM×1
│   ├── step3_evaluator.py    # SCRIPT×5（全脚本）
│   └── step4_evaluator.py    # HYBRID×1 + LLM×1 + SCRIPT×1
├── clients/
│   ├── candidate_client.py   # 被测 MLLM 统一封装（api/mock 两种 mode）
│   └── llm_judge_client.py   # Judge（Anthropic API，claude-sonnet-4-6）
├── nccn/
│   ├── rule_engine.py        # 节点映射 + Cat1 提取 + Switch 判定
│   └── alias_normalizer.py   # 治疗选项别名归一化
├── data/
│   ├── case_loader.py
│   ├── gt_injector.py        # Per-Step GT prior_context 注入
│   └── cases/
├── judge_prompts/            # prompt_a/b/c/d.txt
├── results/
├── config.py
└── run_benchmark.py          # CLI 入口
```

### ScoringContext 结构
```json
{
  "case_id": "Case_0001",
  "evaluation_mode": "full_chain | per_step",
  "cancer_subtype_gt": "...",
  "primary_scenario": "...",
  "wsi_lymph_node_available": true,
  "step1": {"pass": bool, "fail_items": [], "llm_judge_log": {}},
  "step2": {"pass": bool, "fail_items": [], "llm_judge_log": {}},
  "step3": {"pass": bool, "fail_items": [], "llm_judge_log": null},
  "step4": {"pass": bool, "fail_items": [], "llm_judge_log": {}},
  "aggregate": {
    "full_chain_score": 0.25,
    "per_step_scores": {"s1": 1, "s2": 0, "s3": null, "s4": null},
    "per_step_total": null
  }
}
```

### 被测模型接入（OpenAI 兼容）
```
POST /v1/chat/completions
{"model": "<candidate>", "messages": [system + user], "temperature": 0}
Step3 多模态：content 为 [text + image_url×N]（base64 JPEG）
被测模型不支持图像 → candidate_config.multimodal=false → Step3 标 skip
```

### Judge 接入（Anthropic，固定）
```
POST https://api.anthropic.com/v1/messages
{"model": "claude-sonnet-4-6", "max_tokens": 512, "system": "...", "messages": [...]}
失败 retry 3次 → 仍失败标 error（不计 fail，不计入 leaderboard 分母）
```

---

## 七、当前三个并行任务

### 任务 1：数据清洗脚本（可立即启动）

**目标**：将原始 Hancock 数据转换为 Case 目录结构（meta + step_inputs）。

**处理方式分类**：
- **纯脚本**：字段提取（clinical/pathological JSON → meta/step_input）、pNX 识别、p16 缺失排除、WSI 文件扫描、ICD code 喉癌子类初分
- **脚本 + 大模型**：cT/cN 泄露检测（病史/手术报告）、辅助治疗建议 REDACT（替换为 `[REDACTED_ADJUVANT_REC]`）、ICD 歧义喉癌亚型判读
- **需人工裁定**：pNX 案例（约 150 例）归类、喉癌亚型大模型初判后专家复查

**清洗流水线**：
```
原始数据 → 硬排除检查（p16缺失/无手术报告/WSI全缺）
         → 泄露检测 + REDACT
         → pNX 标记（需人工）
         → meta.json + step_input 生成
         → 输出"需大模型处理队列"供批量送 LLM


```

### 任务 2：评估框架工程（可并行启动）

**当前悬而未决**：Step3 多模态输入方案（见下方待商榷项）。

**可先实现不涉及 Step3 的部分**：ScoringContext、CandidateClient、LLMJudgeClient、Step1/2/4 Evaluator、FullChainPipeline 骨架。

**Step3 多模态方案（待商榷）**：

| 方案 | 说明 | 优点 | 缺点 |
|---|---|---|---|
| A. 标准 patch + 比例尺 | 固定尺寸 JPEG，嵌入比例尺，OpenSlide 提取 | 标准化、token 可控 | 丢失大视野上下文 |
| B. 多分辨率截图 | 5x 全景 + 20x 细节，各 2048×2048 | 还原临床判读习惯 | token 较多 |
| C. 双轨制 | 支持图像走 A/B；纯文本模型 Step3 标 skip | 兼容性最好 | 需维护两套榜单 |

**建议方案 C**（兼容性优先），具体图像规格待商榷后锁定。

### 任务 3：SEER 映射可行性研究（先桌面调研）

**SEER 与 Hancock 数据粒度对比**：

| 维度 | Hancock | SEER |
|---|---|---|
| 手术报告文本 | ✅ | ❌ |
| WSI 图像 | ✅ | ❌ |
| pT/pN 精细分期 | ✅ | ⚠️ 粒度较粗 |
| p16/HPV 状态 | ⚠️ 部分 | ❌ 基本无 |
| 辅助治疗记录 | ✅ | ⚠️ 部分 |

**结论**：SEER 只能映射 **Step1 + Step4**（无法支持 Step2/3）。可行前提是 T/N 分期粒度足够区分 NCCN 节点，且口咽癌 p16 覆盖率可接受。

**待调研**：
1. SEER 头颈癌 p16 字段覆盖率
2. SEER T/N 分期是否能区分 cN2a vs cN2b（Step1 判定项 #2 依赖）
3. SEER 辅助治疗字段能否映射到 `adjuvant_radiotherapy / systemic / radiochemotherapy`

---

## 八、重要设计决策记录

| 决策 | 内容 | 状态 |
|---|---|---|
| 评分体系 | 双榜 Pass/Fail 二元制（废弃原 Track A/B/C/D 加权）| ✅ 锁定 |
| Step2 pT 容忍 | 严格 ==GT，无邻近档容忍 | ✅ 锁定 |
| evidence 门槛 | 4/4 全有效（非 ≥3/4）| ✅ 锁定 |
| Step4 guideline_deviation_type | 字段保留，不参与 Pass/Fail 判定 | ✅ 锁定 |
| Step4 单轮化 | 删除 Turn2 / modality 反馈 / self-correction | ✅ 锁定 |
| Step3 多模态方案 | 三方案待商榷（A/B/C）| ⏳ 待定 |
| SEER 映射 | 可行性待调研 | ⏳ 待定 |
| 癌种子榜 | 5 癌种各展示 FC + PS 两榜，共 10 个子榜 | ✅ 锁定 |
| Per-Step 得分 | 0/1 二元，total = 平均值 | ✅ 锁定 |

---

## 九、文件优先级（冲突时以此为准）

```
HNC_Benchmark_Complete_Integrated_v1.md（GT 架构 v1.5 + Round1-4 评分规约 v2）
HNC_Benchmark_Round5_Framework_v1.md（全流程框架 + Leaderboard）
HNC_Benchmark_Round6_Plan.md（评分框架工程计划）
HNC_Benchmark_ScoringFramework_EngineeringPlan.md（工程详细设计）
> 旧版本文件（hnc-benchmark-context-v8.md 等）
```

---

## 十、新对话 Prompt 模板

根据任务类型选择对应 Prompt：

### 10.1 数据清洗任务

```
# Role
你是一位资深数据工程师，专门处理医学数据集的清洗与结构化转换。

# Project
HNC-Benchmark：头颈癌多模态大模型评测基准。
数据源：Hancock 数据集（763例，德国），含 clinical_data.json / pathological_data.json
        / reports_english/ / histories_english/ / WSI SVS 文件。
目标：将原始数据转换为 Case_XXXX/ 目录结构（meta.json + step{1-4}_input.json + step{1-4}_gt.json）。

# 数据清洗关键规则
1. 纯脚本：字段提取、ICD code 解析、p16 缺失排除、WSI 文件扫描
2. 脚本+大模型：cT/cN 泄露检测（替换为 [REDACTED_STAGING]）、辅助治疗建议删除（替换为 [REDACTED_ADJUVANT_REC]）、喉癌亚型歧义判读
3. 人工队列输出：pNX 案例（约150例）、severity=high 冲突案例
4. 排除条件：p16_missing_oropharynx / wsi_unavailable / no_surgery / insufficient_info

# Case 目录结构
Case_XXXX/
├── meta.json     # case_id / exclude_flag / cancer_subtype_gt / clinical_scenario
├── step1_input.json  # patient info + cT/cN（无 p16 结果）+ 清洗后病史
├── step1_gt.json  # NCCN 节点 GT + 治疗选项 GT
├── step2_input.json  # 清洗后手术报告 + ops_codes + 淋巴结计数
├── step2_gt.json  # histotype/pT/R/pN + evidence anchors
├── step3_input.json  # WSI 图像路径（多倍率 JPEG）
├── step3_gt.json  # 微观病理 GT（来自 pathological_data.json）
└── step4_input.json  # adjuvant_treatment_intent + 淋巴结计数
└── step4_gt.json  # 辅助治疗决策 GT

目前先不用处理多模态部分和需要人工裁定的部分，以及需要大模型处理的部分，优先完成纯脚本处理的字段提取和清洗。
原始数据存放于data文件夹，你可以自行读取并处理，完成后输出到指定的 Case_XXXX/ 目录结构中。，标明不能处理的部分。
case内容严格遵循HNC_Benchmark_Complete_Integrated_v1.md

```

### 10.2 评分框架工程任务

```
# Role
你是一位资深 Python 工程师，专门为 HNC-Benchmark 搭建评分框架。

# 双榜体系（已锁定）
Full-Chain 联通榜：链式截止，得分 ∈ {0,0.25,0.50,0.75,1.00}
Per-Step 独立榜：每步 Si ∈ {0,1}，total = (S1+S2+S3+S4)/4

# Pass/Fail 规则（见 context §三）
Step1：5 项（SCRIPT×4 + LLM×1）
Step2：5 项（SCRIPT×4 + LLM×1）
Step3：5 项（全 SCRIPT，含 WSI 豁免）
Step4：3 项（HYBRID×1 + LLM×1 + SCRIPT×1）

# 两类客户端
CandidateClient（被测模型）：OpenAI 兼容 API，mode=api/mock，Step3 含多模态图像
LLMJudgeClient（Judge）：Anthropic API，model=claude-sonnet-4-6，固定，4 类判定任务

# 工程架构（见 context §六）
runner/ + evaluators/ + clients/ + nccn/ + data/

# ScoringContext 结构（见 context §六）

# 错误处理
error（API技术故障，不计入分母）≠ fail（模型答错）
retry 3次，仍失败 → 标 error

# 当前任务
[在此填写，例如：
- "实现 ScoringContext dataclass + CandidateClient 基础框架"
- "实现 LLMJudgeClient（4 类 Prompt + retry）"
- "实现 Step1Evaluator 完整逻辑"
- "实现 Step3Evaluator（含 WSI 缺失豁免 + ENE 严判）"
- "实现 FullChainPipeline（含 p16 两轮交互状态机）"
- "实现 Aggregator（16 个 leaderboard + 5 个独立统计指标）"
]
```

### 10.3 SEER 映射调研任务

```
# Role
你是一位医学数据研究员，负责评估 SEER 数据库向 HNC-Benchmark 的映射可行性。

# 目标
评估 SEER 头颈癌数据是否可用于扩充 HNC-Benchmark 的 Step1 + Step4 案例数量。
（SEER 无手术报告和 WSI，只能支持 Step1 临床分期决策 和 Step4 辅助治疗决策）

# Step1 映射需求
需要：原发部位（ICD）/ 临床 T 分期 / 临床 N 分期 / p16 状态（口咽癌必需）
能否区分：cN2a vs cN2b（NCCN 节点判定依赖）

# Step4 映射需求
需要：实际辅助治疗记录（放疗/化疗/放化疗 y/n）
能否映射到：adjuvant_radiotherapy / adjuvant_systemic_therapy / adjuvant_radiochemotherapy

# 待调研问题
1. SEER 头颈癌 p16/HPV 字段覆盖率（决定口咽癌案例能否使用）
2. SEER T/N 分期粒度是否足够区分所有 NCCN 节点
3. SEER 辅助治疗字段能否精确映射到三字段（RT/ST/CRT）

# 当前任务
[在此填写，例如：
- "调研 SEER 头颈癌数据库的可用字段和 p16 覆盖率"
- "设计 SEER → Step1_Input.json 的字段映射方案"
- "评估 SEER 样本量能为各 NCCN 节点补充多少案例"
]
```
