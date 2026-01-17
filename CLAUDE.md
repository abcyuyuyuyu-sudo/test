**Purpose（工作手册目标）**

- 本文件为 `libs-coal-online-quant-v3/` 仓库提供项目级持久上下文与执行规约，面向交互式代码助手（Claude/ChatGPT/本地 LLM）。
- 目标：在 **V3 复杂现场版标准** 下，确保所有工作产出满足：**可复现、可审计、可回滚、可验收**。
- 约束基线：本项目只做“胶水层编排与集成”，严禁重复实现成熟库功能；任何不确定信息必须标注 `Unknown` 并给出需补齐证据清单。

**Project Scope（范围与边界）**

- **In-Scope（允许）**
  - 光谱处理算法链路的业务编排、模块组合调度、参数配置与输入输出适配（位于 `src/libscoal/*` 与 `scripts/*`）。
  - 离线评估与回放一致性（10–20 Hz → 1 Hz）、化验对齐评估、回归测试、发布/回滚、漂移闭环（按 `docs/` 与 `configs/`）。
  - 资产注册与审计事件记录（`src/libscoal/registry/*`、`src/libscoal/ops/*`）。
- **Out-of-Scope（禁止）**
  - 通信/下位机/时序控制器实现细节；只能用接口字段假设描述。
  - 激光、粉尘等危险操作步骤细节；仅可给风险提示与“需资质复核/按机构 SOP”。
  - 任何编造实验精度/结果/引用。
  - 将帧级（frame-level）数据当作独立训练样本。

**Repository Map（目录与入口：先看这些）**

- `docs/`（唯一口径来源；冲突时以此为准）
  - `docs/data_dictionary.md`：HDF5 字段字典（**唯一真源**）
  - `docs/design/architecture_v3.md`：V3 架构（状态机/漂移闭环/发布回滚）
  - `docs/design/interfaces.md`：Raw/Preprocessed/Inference/Fusion 接口字段
  - `docs/protocol_alignment.md`：化验对齐（批次/延迟 Δ/防泄漏）
  - `docs/protocol_evaluation.md`：FAT/SAT/回归口径与指标
  - `docs/ops/failure_handling_matrix.md`：失败分类 → 策略矩阵
  - `docs/ops/recalibration_sop.md`：复标/再校准 SOP（仅算法与数据流程）
  - `docs/ops/release_process.md`：发布/回归/回滚流程
- `configs/`（所有行为必须可配置、可 hash、可注册）
  - 数据：`dataset.yaml`, `h5_schema.yaml`
  - 预处理：`preprocess.yaml`, `loi_registry.yaml`
  - Step1/2：`model_plsr.yaml`, `acq_config.yaml`
  - 不确定度：`uncertainty.yaml`
  - 融合/对齐：`fusion_1hz.yaml`, `alignment.yaml`
  - V3：`drift.yaml`, `failure_handling.yaml`, `scoring.yaml`, `release.yaml`, `qa_threshold_set.yaml`
- `scripts/`（官方可重复入口；Notebook 仅辅助）
  - `00_validate_csv.py` → `01_build_h5_from_csv.py` → `02_compute_cell_mean.py` → `03_step1_focus_energy.py` → `04_step2_channel_delay.py` → `05_calibrate_qa_thresholds.py` → `06_train_plsr_models.py` → `07_evaluate_protocol.py` → `08_offline_replay.py` → `09_export_deploy_bundle.py` → `10_build_golden_set.py` → `11_run_regression.py` → `12_build_release.py` → `13_rollback_release.py` → `14_recalibrate_thresholds.py`
- `src/libscoal/`（只负责胶水编排与适配，不重复实现成熟库）
  - 数据：`dataio/*`；预处理：`preprocess/*`；质量：`quality/*`；调优：`tuning/*`；融合：`fusion/*`；对齐：`alignment/*`；漂移：`drift/*`；运行态：`ops/*`；注册表：`registry/*`
- `tests/`（发布前硬门禁）
  - 单元：`test_*`
  - 回归：`tests/regression/*`（golden set + release diff budget）
- `data/`、`models/`、`logs/` 通常 Git 忽略；`artifacts/` 建议保留报告（至少 `artifacts/reports/`、`artifacts/regression/`）。

**Non-Negotiables（V3 硬性要求：任何工作不得违反）**

- **状态机字段必须输出**：`state=nominal|degraded|reject`，且必须包含 `staleness_sec`、`completeness`。降级/拒识必须给原因。
- **FailureHandling**：必须定义失败分类 → 策略矩阵；任何进入 `degraded/reject/rollback` 必须写审计事件：`event_id/reason/release_id/threshold_set_id/acq_config_id`（以及 pipeline/model 等）。
- **漂移闭环**：必须定义漂移指标（`spec_shift/embedding_ood/threshold_drift`）、触发条件、动作优先级（阈值再校准 → 校正 → 重训）与恢复条件。
- **回归测试**：必须固化 `golden set + splits`；任何 `model/threshold/preprocess` 变更必须跑回归并出对比报告。
- **Release Registry**：必须维护 `release_id`，绑定模型与两套资产（`acq_config` 与 `qa_threshold_set`）及全部 hash；必须支持回滚。
- **评分量表**：必须输出硬门禁 + 加权评分（Go/No-Go），用于版本对比与发布决策。
- **仍然禁止**：帧级 CV 当独立样本、编造精度、写密钥/个人数据、输出危险操作步骤。

**Data Contract（数据与样本口径：必须按此执行）**

- 项目：LIBS 煤炭在线定量监测（仅光谱）。
- 节拍：10–20 Hz 光谱推理缓存；1 Hz 输出：灰分/硫分/热值 + 置信度/区间 + 异常/拒识 + 数据完整性 + 状态。
- 数据来源：CSV；每（样品 × 焦距 × 能量 × 延时）无重复；每 cell 350 帧，去掉前 50 帧；合并为带元数据与校验的 HDF5。
- **训练/评估样本**：`cell_mean` 与 `cell_stats`（帧仅用于稳定性与阈值/漂移校准；不得作为独立训练样本）。

**Standard Execution Order（标准运行顺序：按脚本；每步必须产出工件）**

> 任何偏离必须在报告与 registry 中记录，并给出理由与影响范围。

- **Step 0：CSV 预检**
  - 入口：`scripts/00_validate_csv.py`
  - 产出：`artifacts/reports/ingest/<dataset_id>_validate_csv.md`（重复 cell、缺列、帧计数、元数据映射）
- **Step 1：CSV → HDF5（含 meta 与 schema 校验）**
  - 入口：`scripts/01_build_h5_from_csv.py`
  - 配置：`configs/dataset.yaml`, `configs/h5_schema.yaml`
  - 产出（必须）：
    - `data/h5/<dataset_id>.h5`（或 `artifacts/hdf5/`，以实际为准）
    - `artifacts/reports/ingest/<dataset_id>_ingest_report.md`
    - `artifacts/tables/<dataset_id>_manifest.json`（含 hash）
- **Step 2：cell_mean/cell_stats（去前 50 帧）**
  - 入口：`scripts/02_compute_cell_mean.py`
  - 产出（必须）：
    - `data/derived/<dataset_id>_cell_mean.*`
    - `data/derived/<dataset_id>_cell_stats.*`
    - `artifacts/reports/features/<dataset_id>_cell_stats_report.md`
- **Step 3：Step1（全谱 PLSR 选焦距+能量）**
  - 入口：`scripts/03_step1_focus_energy.py`
  - 配置：`configs/model_plsr.yaml`, `configs/acq_config.yaml`
  - 产出（必须）：
    - `artifacts/tables/step1/<dataset_id>_plsr_rank_table.csv`
    - `artifacts/reports/step1/<dataset_id>_step1_report.md`
- **Step 4：Step2（单通道延时调优，多目标+敏感性；每通道一个共用最优延时）**
  - 入口：`scripts/04_step2_channel_delay.py`
  - 产出（必须）：
    - `artifacts/tables/step2/<dataset_id>_delay_tradeoff_table.csv`
    - `configs/acq_config.yaml` 更新（或生成版本化副本）并产生新 hash
    - `artifacts/reports/step2/<dataset_id>_step2_report.md`
- **Step 5：Step3（预处理+双层质量门控+阈值校准）**
  - 入口：`scripts/05_calibrate_qa_thresholds.py`
  - 配置：`configs/preprocess.yaml`, `configs/qa_threshold_set.yaml`, `configs/failure_handling.yaml`
  - 产出（必须）：
    - `models/exports/thresholds/<threshold_set_id>/`（或 `artifacts/`，以实际为准）
    - `artifacts/reports/qa/<threshold_set_id>_calibration_report.md`
    - 质量门控样例输出（含 flags/OOD 分数/映射逻辑的可追溯说明）
- **Step 6：训练/导出（如启用）**
  - 入口：`scripts/06_train_plsr_models.py`, `scripts/09_export_deploy_bundle.py`
  - 产出（必须）：模型导出包（含模型、configs、hash 清单）
- **Step 7：协议评估（FAT/SAT）**
  - 入口：`scripts/07_evaluate_protocol.py`
  - 配置：`docs/protocol_evaluation.md`, `configs/scoring.yaml`
  - 产出（必须）：`artifacts/reports/eval/<eval_id>_protocol_report.md`
- **Step 8：离线回放（10–20Hz→1Hz；时延/完整性/一致性；化验对齐评估）**
  - 入口：`scripts/08_offline_replay.py`
  - 配置：`configs/fusion_1hz.yaml`, `configs/alignment.yaml`
  - 产出（必须）：
    - `artifacts/reports/replay/<run_id>_offline_replay.md`
    - `artifacts/tables/replay/<run_id>_predictions_1hz.*`（必须含状态机字段）
- **V3 Step 9：固化 golden set + splits**
  - 入口：`scripts/10_build_golden_set.py`
  - 产出（必须）：
    - `data/golden/<golden_set_id>/manifest.json`
    - `data/splits/<split_id>.json`（或等价）
    - `artifacts/reports/regression/<golden_set_id>_definition.md`
- **V3 Step 10：回归测试（对比上一 release）**
  - 入口：`scripts/11_run_regression.py`
  - 产出（必须）：
    - `artifacts/regression/<run_id>/metrics.json`
    - `artifacts/reports/regression/<run_id>_diff_report.md`
    - `artifacts/tables/regression/<run_id>_scorecard.csv`
- **V3 Step 11：Build Release（release_id + registry + 评分表）**
  - 入口：`scripts/12_build_release.py`
  - 产出（必须）：
    - `models/registry/release_registry.(json|yaml)` 更新（或新增条目）
    - `artifacts/reports/release/<release_id>_release_notes.md`
- **V3 Step 12：Rollback（按 release_id 回滚）**
  - 入口：`scripts/13_rollback_release.py`
  - 必须写审计事件：`rollback_entry` + 原因 + 指向回滚目标 release。
- **V3 Step 13：漂移触发后的阈值再校准入口**
  - 入口：`scripts/14_recalibrate_thresholds.py`
  - 需遵守漂移闭环动作优先级，并在回归/发布前完成评分与 registry 更新。

**Output Contract（1 Hz 输出字段：必须包含；与 `docs/design/interfaces.md` 一致）**

- 必填：
  - `state`（nominal/degraded/reject）
  - `staleness_sec`、`completeness`
  - `pred`（灰/硫/热值）
  - `conf`（置信度/区间；必须引用 `configs/uncertainty.yaml` 的配置版本）
  - `flags`（解释型门控输出）与 `ood_score`（统计型 OOD 输出）
  - `data_integrity`（schema/hash/缺字段）
  - 追溯字段：`release_id`, `threshold_set_id`, `acq_config_id`, `pipeline_hash`, `model_hash`, `split_id`, `golden_set_id`
  - 降级/拒识：`reasons[]` 非空，并写审计事件（见下）

**Audit（审计事件：进入 degraded/reject/rollback 必须记录）**

- 事件必须含：`event_id, timestamp, reason, release_id, threshold_set_id, acq_config_id`（以及 pipeline/model 等）
- 事件位置：优先写入 `artifacts/`（避免污染核心代码区），并在 `models/registry/` 的 release 条目中引用事件范围。
- 审计要求：不得模糊判断；必须逐项给出可验证证据（引用具体路径/配置 hash/测试结果）。

**Development Principles（开发约束：胶水开发 + 通用开发 + 系统性完整性检查）**

- **胶水开发约束（必须执行）**
  - 不得自行实现底层/通用逻辑；必须优先、直接、完整复用既有成熟仓库与生产级库。
  - 不得复制依赖库代码进本项目；不得裁剪/重写/降级封装依赖库。
  - 依赖路径必须真实存在且指向完整实现；必须防止路径遮蔽/重名模块/隐式 fallback。
  - 不得写 Mock/Stub/Demo 冒充真实实现；不得存在占位实现、空逻辑或“先写接口后补实现”。
  - 本项目仅承担业务流程编排、模块组合调度、参数配置与 I/O 适配。
  - 生成代码时必须标注：哪些能力来自外部依赖（名称/模块路径/版本）。
- **通用开发约束（节选关键条款）**
  - 不得补丁式只修局部忽视整体；不得引入不必要的中间状态；不得跳过验证；不得吞异常；不得臆测接口行为。
  - 必须保持结构简单清晰、注释充分；遵循 SOLID/DRY；不清楚必须明确说明并留痕。
- **系统性完整性审计要求（必须执行）**
  - 必须核查导入路径真实、完整且生效；排除路径遮蔽与重名误导加载。
  - 所有审计结论必须基于可验证的代码与路径分析；不得输出主观推测。

**Change Rules（变更规范：改什么必须同步改什么）**

- 改 `configs/*`：必须产生新 hash，并在报告与 registry 中记录；不得“静默改配置”。
- 改 `src/libscoal/*`：必须更新 `tests/*`（至少新增/更新相关测试）并更新 `docs/`（涉及口径/接口/协议）。
- 改 `quality gating/threshold`：必须重新校准，产出校准报告；必须生成/更新 `threshold_set_id` 并注册到 release。
- 改 `preprocess`：必须跑回归，产出 diff report 与 scorecard；硬门禁不通过不得发布。
- 任何变更必须能够被 `scripts/*` 一键复现（建议配合 `Makefile`）。

**Common Pitfalls（常见坑：你必须主动提醒并防止）**

- 帧泄漏：帧当样本导致虚假“高精度”。
- 资产不分离：模型/阈值/采集配置/预处理未绑定 release，导致不可回滚。
- 阈值未校准就比较：直接导致 reject/degraded 口径失真。
- 未固化 splits/golden set：回归不可比，发布不可审计。
- 未按协议输出状态机字段：现场不可观测不可追责。
- 依赖库被“本地重写/裁剪版”替代：属于重大合规/质量事故。

**Minimal Executable V3 Flow（最小可执行：漂移 → 再校准/重训 → 回归 → 发布 → 回滚）**

- 触发：漂移指标超限 / 维护事件 / 拒识率上升 → 写审计事件 `drift_triggered`
- 动作优先级：阈值再校准（`14_recalibrate_thresholds.py`）→ 校正 → 重训（如启用）
- 必跑：`11_run_regression.py` → 生成 diff report + scorecard
- Go/No-Go：硬门禁通过且评分达标 → `12_build_release.py` 写 registry
- 否则：停止发布，建议回滚 → `13_rollback_release.py` 并写 `rollback_entry`

**How to Work With Me（你与我交互时我期望的输入/输出）**

- 你给我任务时，至少提供：目标脚本/模块、涉及配置、预期工件、验收标准（若未知可 `Unknown`）。
- 我给你输出时，必须包含：改动范围、触发哪些测试/回归、生成哪些工件、如何验收、如何回滚、依赖库复用声明（若涉及）。
