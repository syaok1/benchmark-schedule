# HNC-Benchmark 项目上下文摘要
# 版本：summary-v1.0 | 2026-04-05
# 用途：在新对话中快速恢复项目上下文

---

## 0. 项目概述

**目标**：构建评测专业/非专业多模态大语言模型（MLLM）在头颈癌临床推理能力的 Benchmark，做成公开网站（支持本地部署 + GitHub Pages）。

**数据集**：Hancock（https://www.cancerimagingarchive.net/collection/hancock/）
- 763例，德国Erlangen单中心，2005–2019，CC BY 4.0 公开

**指南**：NCCN Head and Neck Cancers 2026.V1

**开发角色分工**：
- 开发者（我）：搭建框架、编写所有脚本，本地无数据
- 数据运行方：下载 Hancock 数据、运行脚本、生成数据集、跑评测
- 所有脚本必须支持 `--dry-run`，数据路径通过配置文件传入，不硬编码

---

## 1. 核心设计决策（已确认，不再修改）

### 1.1 四步推理流程（原五步，Step1/2已合并）

| 步骤 | 阶段名 | 输入 | 核心输出 |
|------|--------|------|---------|
| Step 1 | STAGING_AND_TREATMENT_DECISION | 基本信息 + **直接给出的 cT/cN** + 病史 | NCCN路径节点 + 推荐治疗方案 |
| Step 2 | SURGERY_GROSS_PATHOLOGY | 手术报告文本（已清洗）| 初步 pT/R/histotype + Evidence Anchor |
| Step 3 | WSI_MICROSCOPIC_CONFIRMATION | WSI patches（原发灶5x/20x + 淋巴结10x）| 更新 pT/pN/R + 微观特征 + 浸润深度 |
| Step 4 | ADJUVANT_TREATMENT_DECISION | Steps 1-3 模型输出汇总 | 辅助放化疗决策 + reasoning |

**步骤权重**：Step1=25%, Step2=20%, Step3=30%, Step4=25%

**重要**：describe_focus（影像学描述合成）已删除。cT/cN **直接**来自结构化数据，但需先确认 Hancock 数据集中 cT/cN 字段的实际来源（见第6节待确认问题）。

### 1.2 口咽癌 p16 附加子任务

- Step 1 中口咽癌患者：模型必须先声明"需要 p16 结果"，系统再提供，模型给出最终路径
- 独立评分，不计入总分
- 531号患者（p16未测，GT稀少）建议剔除

### 1.3 双轨评分

- **AR Score**：级联传播（每步用模型自身上一步输出）
- **IND Score**：每步注入 GT（独立评分）

### 1.4 非手术患者

- `is_non_surgical: true` → Step 1 后直接跳 Step 4，Step 2/3 标记 NA

---

## 2. 评分体系

### 总分结构
```
Case总分 = Σ(步骤权重 × 步骤分) × safety_multiplier
步骤分 = 60% × 结构化字段 + 40% × LLM-Judge
Step4 = 20% × 汇总子任务 + 80% × 决策子任务
safety_multiplier = max(0.5, 1.0 - critical_error_count × 0.2)
```

### 结构化字段评分规则

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

### 临床安全分（CSS，独立报告）

| Flag | 触发条件 | 惩罚 |
|------|---------|------|
| PERINODAL_MISS | 漏报perinodal invasion | Step3×0.3；CSS-0.30 |
| CHEMORT_MISSED | 漏开放化疗 | Step4×0.2；CSS-0.30 |
| TEMPORAL_VIOLATION | 阶段时序混淆 | 违规字段清零 |
| HALLUCINATION_FLAG | 幻觉引用（evidence无原文支持）| evidence字段清零；CSS-0.10 |
| LN_WSI_HALLUC | 无WSI仍输出yes/no | 该字段×0.2；CSS-0.10 |
| CASCADE_ERROR | 升期后路径未切换 | 额外-0.15 |

```
CSS = 1.0 - (高危漏报×0.30) - (幻觉次数×0.10)
```

### LLM-Judge
- 每字段按3条标准逐一T/F：①无幻觉 ②逻辑可追溯 ③术语准确
- 调用 claude-sonnet-4-6，每字段2次取多数

---

## 3. 数据集关键事实

### Hancock 数据模态

| 模态 | 数量 | 可用率 | 格式 |
|------|------|--------|------|
| 原发灶 WSI | 709张 | 92% | SVS，0.1213µm/px |
| 淋巴结 WSI | 396张 | 52% | SVS |
| GeoJSON 肿瘤标注 | ~701个 | ~100% | GeoJSON |
| 手术报告（英译） | >98% | >98% | .txt |
| 病理结构化数据 | 763例 | 100% | JSON（GT核心来源）|
| 临床结构化数据 | 763例 | 100% | JSON |

### 必须预处理
- 手术报告中医生给出的辅助治疗建议 → **必须删去**
- 病史/报告中透露 cT/cN 的内容 → **标注 `contains_cTcN_leak: true` 并删去**
- 531号患者（口咽癌，p16未测）→ **建议剔除**
- 声门/声门上喉癌在 Hancock 中均归为"喉癌" → **须按 ICD codes 重新分类**

### 淋巴结 WSI 缺失处理

| 情况 | meta标记 | 模型应输出 | 评分 |
|------|---------|-----------|------|
| pN0 | `ln_wsi_missing_reason: "pN0"` | `"not_applicable"` | 满分 |
| pN+ 但无WSI | `ln_wsi_missing_reason: "specimen_unavailable"` | `"Cannot Determine"` | 满分；输出yes/no扣分+CSS |
| pNX（未做颈清） | `pNX_no_neck_dissection` | `"pNX"` + `perinodal: "not_applicable"` | 精确满分；输出pN0/pN+视为幻觉 |
| pNX（记录缺失） | `pNX_data_missing` | 建议排除评测集 | — |

---

## 4. 专家标注需求

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

## 5. NCCN路径节点映射（规则引擎核心，不得修改）

### 口腔癌
- T1-2N0 → OR-2
- T3N0 / T1-3N1-3 / T4aN0-3 → OR-3
- T4b/无法切除 → ADV-1 | M1 → ADV-2

### 口咽癌 p16-
- T1-2N0-1 → ORPH-2
- T3-4aN0-1 → ORPH-3
- T1-4aN2-3 → ORPH-4
- T4b → ADV-1 | M1 → ADV-2

### 口咽癌 p16+
- M1 → ADV-2
- T1-2N0 → ORPHPV-1
- T0-2N1（单节点≤3cm）→ ORPHPV-2
- T0-2N1（>3cm或同侧≥2个≤6cm）/ T1-2N2 / T3N0-2 → ORPHPV-3
- T0-3N3 / T4N0-3 → ORPHPV-4

### 下咽癌（p16无临床意义，不要求）
- 适合喉保留（多数T1N0，部分T2N0）→ HYPO-2
- T1-3N0-3 → HYPO-3
- T4aN0-3 → HYPO-5
- T4b/无法切除 → ADV-1 | M1 → ADV-2

### 声门喉癌
- 原位癌/适合保留喉（T1-2N0，选定T3N0）→ GLOT-2
- T3全喉N0-1 → GLOT-3
- T3全喉N2-3 → GLOT-4
- T4a → GLOT-6
- T4b → ADV-1 | M1 → ADV-2

### 声门上喉癌
- 适合保留喉（多数T1-2N0，部分T3）→ SUPRA-2
- T3N0全喉 → SUPRA-3 | T4aN0 → SUPRA-8
- N+ → SUPRA-4分流：
  - T1-2N+ / 部分T3N1 → SUPRA-5
  - 多数T3N1-3 → SUPRA-6
  - T4aN1-3 → SUPRA-8
- T4b → ADV-1 | M1 → ADV-2

**完整治疗方案详见 hnc-benchmark-context-v5.md 第6节**

---

## 6. 辅助治疗决策（规则引擎核心，不得修改）

### 不良特征分级
- 🔴 高危（1类）：perinodal=yes 和/或 R1/margin<0.1cm → 系统治疗+放疗
- 🟡 其他风险（2A类）：pT3/T4、pN2/N3、多LN+、LVI+、Pn+、单LN+无其他 → 放疗或考虑系统治疗/放疗
- 🟢 无不良特征：pN0或pN1无其他风险 → 观察或考虑RT

**各癌种节点详细决策流详见 hnc-benchmark-context-v5.md 第7节**

---

## 7. 特殊案例处理

| 情况 | meta标记 | 处理 |
|------|---------|------|
| 非手术首选（本数据集没有） | `is_non_surgical: true` | Step1后直接跳Step4，Step2/3标记NA |
| 声门/声门上区分 | `icd_codes_larynx_subtype` | AI按ICD归类，专家全量检查 |
| 病史透露cTcN | `contains_cTcN_leak: true` | 删除泄露内容，作为信息更新能力测试 |
| 患者拒绝辅助 | `patient_declined: true` | 按guideline_recommends字段评分 |
| 数据矛盾患者 | `data_contradiction: true` | 专家修正后提供正确GT |
| 531号口咽癌p16未测 | 建议剔除 | 不纳入评测集 |

pNX(20%数据缺失)
| 场景 | cN（术前| pN（术后病理）|说明|
|------|---------|------|------|
|正常案例|cN0/cN1/cN2/cN3|pN0/pN1/pN2/pN3|正常处理|
|未做淋巴颈清|cN0|pNX|术前评估阴性，未做颈清，无病理结果|
|颈清但未评估|cN+|pNX|数据缺失，建议排除或另设难度层次问题检查是否存在幻觉|

---

## 8. 案例数据结构

```
dataset/cases/Case_{patient_id}/
├── meta.json
├── step1_input.json     # 基本信息 + cT/cN（来源待确认，见第9节）
├── step1_gt.json        # NCCN路径GT + p16子任务GT
├── step2_input.json     # 手术报告（已清洗）+ 病史 + 淋巴结数量
├── step2_gt.json        # 初步pT/R/histotype + Evidence Anchors
├── step3_images/
│   ├── primary_5x_scalebar.jpg
│   ├── primary_20x_scalebar.jpg
│   └── lymphnode_10x_scalebar.jpg
├── step3_gt.json        # 微观病理GT（来自pathological_data.json）
└── step4_gt.json        # 辅助治疗GT
```

---

## 9. ⚠️ 待确认的关键问题（影响后续开发）

**Step 1（初始治疗决策）的输出格式**：关于nccn指南给出的手术内容，商榷是需要将其单独列出（需考虑篇幅是否过长），还是应当和治疗选项（options）放在一起以符合指南的格式。——————讨论结果：像指南一样放在一起
**Step 3（WSI 切片更新）的输入内容**：商榷该步骤的具体输入物，包括带比例尺的原发灶全景图（5x）和细节图（20x）、带比例尺的淋巴结图（10x），以及 Step 2 的模型输出摘要。——————讨论结果：技术问题单独考虑
**Step 3（WSI 切片更新）的影像格式处理**：商榷如何实际向模型输入大像素的 svs 切片影像。——————讨论结果：技术问题单独考虑
**Step 4（辅助治疗决策）的子任务设置**：商榷“子任务 A”（要求汇总 Steps 1-3 的全部信息，输出包含所有 Cannot Determine 字段的完整患者画像）是否具有必要性。——————有没有都行
**评分体系**：整个第 8 节“评分体系”的具体评判细则尚未完全敲定，需要进一步商榷。

---

## 10. AI 辅助清洗模块

### 需要 AI 的三个任务

| 任务 | 脚本 | AI做什么 |
|------|------|---------|
| Evidence Anchor 提取 | `extract_evidence_anchors.py` | 从手术报告定位支持pT/R/histotype的关键原文短语 |
| 辅助治疗建议检测 | `clean_surgery_reports.py` | 识别报告中医生给出的辅助治疗建议段落 |
| cTcN 泄露检测（AI兜底）| `detect_cTcN_leak.py` | 规则匹配未命中时，AI识别隐含的临床分期描述 |
- 其他数据清洗也可由大模型支持

### 统一 AI 接口（ai_client.py）

支持的 provider（config.yaml 中指定）：
- `anthropic` → Anthropic API
- `openai` → OpenAI API
- `deepseek` → DeepSeek API（自定义 base_url）
- `local` → ollama/vllm（兼容 OpenAI 格式）

**切换模型只改 config.yaml，脚本代码不动。**

所有 AI 调用记录到 `logs/ai_calls.jsonl`（timestamp, patient_id, task, prompt, response, duration_ms）。

---

## 11. 项目目录结构

```
hnc-benchmark/
├── CLAUDE.md                          # Claude Code 自动读取
├── README.md
├── config.example.yaml
├── hnc-benchmark-context-v5.md        # 完整项目上下文
├── hnc-benchmark-todo-v3.md           # 完整开发TODO
│
├── data_processing/
│   ├── ai_client.py                   # 统一AI调用接口
│   ├── parse_hancock.py               # P0-2
│   ├── classify_larynx_icd.py         # P0-2
│   ├── detect_data_issues.py          # P0-2
│   ├── detect_cTcN_leak.py            # P0-2（规则+AI兜底）
│   ├── nccn_router.py                 # P0-3（纯规则）
│   ├── adjuvant_router.py             # P0-3（纯规则）
│   ├── clean_surgery_reports.py       # P0-4（AI辅助）
│   ├── extract_wsi_patches.py         # P0-5
│   ├── record_ln_wsi_status.py        # P0-5
│   ├── case_builder.py                # P0-6（主入口）
│   ├── extract_evidence_anchors.py    # P0-6（AI辅助，从case_builder拆出）
│   ├── generate_expert_review.py      # P0-6
│   └── validate_dataset.py            # P0-7
│
├── tests/
│   ├── test_nccn_router.py            # ✓已完成
│   ├── test_adjuvant_router.py        # ✓已完成
│   └── test_structured_scorer.py
│
├── evaluation/
│   ├── prompt_builder.py
│   ├── model_runner.py
│   ├── wsi_image_encoder.py
│   ├── structured_scorer.py
│   ├── llm_judge.py
│   ├── safety_checker.py
│   ├── case_evaluator.py
│   ├── run_benchmark.py
│   └── results_aggregator.py
│
├── docs/                              # GitHub Pages 网站
│   ├── index.html
│   ├── leaderboard.html
│   ├── dataset.html
│   ├── docs.html
│   └── data/leaderboard.json
│
├── .github/
│   ├── workflows/update_leaderboard.yml
│   └── ISSUE_TEMPLATE/submit_eval.md
│
├── dataset/cases/                     # 由他人运行生成
├── expert_review/                     # 由他人运行生成
└── logs/ai_calls.jsonl                # AI调用日志
```

---

## 12. 开发进度

| 模块 | 状态 |
|------|------|
| Session 1：项目骨架 | ⬜ 待开始 |
| Session 2：规则引擎（nccn_router + adjuvant_router + 单元测试）| ⬜ 待开始|
| Session 3：数据处理脚本（P0-2 ~ P0-7）| ⬜ 待开始 |
| Session 4：评测引擎（PHASE 1）| ⬜ 待开始 |
| Session 5：GitHub Pages 网站 | ⬜ 待开始 |

---

## 13. Claude Code 开场 Prompt 模板

```
读取 CLAUDE.md。

重要背景：
- 我负责搭建框架和编写所有脚本，Hancock 数据由他人运行
- 所有脚本必须支持 --dry-run 模式（内置 mock 数据）
- 数据路径通过配置文件或命令行参数传入，不硬编码

当前任务：[填写具体任务]
```

---

## 14. 竞品对比（CancerGUIDE）

**CancerGUIDE**（arXiv:2509.07325）：121例真实NSCLC，13名肿瘤科医生标注，130+小时，数据不公开，单步评测（EHR→路径），无多模态。

**我们的差异化**：
- ✅ 数据完全公开（CC BY 4.0）
- ✅ 四步序列推理（错误级联追踪）
- ✅ 多模态（WSI病理图像）
- ✅ 临床安全评分（CSS）
- ✅ 幻觉检测（Evidence Anchor零幻觉约束）
- ✅ 六种头颈癌亚型（vs 单一NSCLC）

**定位**：CancerGUIDE 解决评测规模化问题；我们解决评测深度和真实性问题。

**避嫌**：不声称"第一个用真实患者数据做NCCN Benchmark"；不用"treatment trajectory"作为主要评分框架；不在NSCLC上建benchmark。

---

*摘要结束 | 完整细节见 hnc-benchmark-context-v5.md 和 hnc-benchmark-todo-v3.md*
