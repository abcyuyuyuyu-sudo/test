# README — LIBS 煤炭在线定量监测（仅光谱）V3（复杂现场版）

**一句话说明**
- 在强扰动现场工况（粉尘/振动/带速/粒度/含水）下，基于 **LIBS 光谱**实现 **10–20 Hz 单谱处理**与 **1 Hz 灰分/硫分/热值**在线输出，并提供**置信度、异常/拒识、漂移监测、回滚与复标闭环**（V3）。

**本仓库解决什么问题**
- 把“离线实验 + 参数调优 + 在线推理 + 质量门控 + 化验对齐 + 漂移复标”工程化为可执行、可审计、可回滚的一套流程与资产体系。
- 当前版本仅依赖光谱；未来等离子体图像将作为“占位接口”引入，不影响当前链路成立。

---

**核心特性（V3 复杂现场版）**
- **双节拍链路**
  - 10–20 Hz：预处理 → 质量门控（规则 + OOD）→ 单谱推理 → 不确定度/拒识 → 缓存
  - 1 Hz：窗口融合（稳健聚合 + 权重）→ 输出灰/硫/热值 + 置信度 + 状态（nominal/degraded/reject）+ 完整性指标
- **强扰动鲁棒性**
  - 用“可用性（Availability）+ 稳定性（Stability）+ 误差（Error）+ 延时敏感性（Sensitivity）”统一评价采集与算法配置
- **参数优化闭环（你现有数据可直接落地）**
  - Step1：全谱 PLSR 选 **焦距×能量**
  - Step2：分通道延时调优（门宽固定）+ 谱线驱动点云筛查（帧/Cell 两层）
  - Step3：质量门控阈值集（qa_threshold_set）校准与版本化
- **漂移闭环与发布治理（V3）**
  - 漂移监测 → 触发再校准/回滚 → 回归门禁（golden set）→ 发布 release_id
- **资产分离与版本追溯（必须）**
  - `acq_config`（采集/门控参数最优配置）与 `qa_threshold_set`（阈值集）分离、分别版本化
  - 所有输出绑定 `data_snapshot_id / preprocess_hash / feature_spec_hash / splits_id / release_id`

---

**名词与数据单位（务必先读）**
- **frame**：单条光谱（10–20 Hz 的一帧；或多脉冲聚合后的结果）
- **cell**：固定组合（样品×焦距×能量×延时×通道）的一组帧（你现在每格 350 帧，去前 50 后用 300）
- **cell_mean**：同一 cell 内帧级预处理后，稳健聚合得到的代表谱（用于训练/评估基本样本）
- **cell_stats**：同一 cell 的稳定性/可用性统计（CV/方差/饱和率/线可检测率等）
- **delay（延时）/ gate_width（门宽）**：门控采集参数；V3 支持“分通道延时最优”，门宽在你当前数据中固定 20 µs

---

**目录结构（完整工程骨架）**
- 代码与配置主路径：
  - `src/`：算法模块（预处理/特征/门控/OOD/融合/漂移/资产注册）
  - `scripts/`：可复现流水线脚本（CSV→HDF5→Step1→Step2→点云→Step3→训练→发布→回归门禁）
  - `configs/`：版本化配置（acq_config / preprocess / LOI&feature_spec / qa_threshold_set / splits / alignment / release_registry）
- 文档与验收：
  - `docs/`：数据字典、接口、对齐协议、评估协议、运维SOP、失败处理矩阵
  - `docs/reports/`：报告模板（数据快照、Step1/2/3、点云、训练、影子模式、回归门禁、漂移复标、失败复盘）
- 数据与产物（默认 Git 忽略或 LFS/外置存储）：
  - `data/`：raw CSV、HDF5、derived（cell_mean/cell_stats）、snapshots
  - `artifacts/`：pointcloud、threshold 校准产物、release bundle（模型+冻结配置+hash+回归报告）
- 工程质量：
  - `tests/`：schema/解析/切分防泄漏/门控/融合/注册表一致性测试
  - `logs/`：pipeline 与 shadow mode 日志

---

**安装与环境（建议最小化）**
- **Python**：建议 3.10+（以 requirements.txt 为准）
- **依赖安装**
  - `pip install -r requirements.txt`
- **环境变量**
  - 复制 `.env.example` → `.env`，至少填：
    - `DATA_DIR`（原始CSV根目录）
    - `HDF5_DIR`（合并后HDF5目录）
    - `ARTIFACT_DIR`（产物目录）
    - `LOG_DIR`（日志目录）

---

**数据准备（你当前数据的“标准入口”）**
- **输入数据形态**
  - CSV：每文件包含若干条光谱（强度数组 + 可选波长轴/表头字段）
  - 组合空间：样品×焦距×能量×延时×通道；每格 350 条；去前 50 条
  - 标签：灰分/硫分/热值（化验；粒度与延迟需在 alignment_config 中声明）
- **强制要求**
  - CSV 文件名或表头必须能解析出：`sample_id, focus_mm, energy_mj, gate_delay_us, channel_id, frame_idx`
  - 解析规则以 `src/data/csv_parser.py` 为唯一实现；字段定义以 `docs/data_dictionary.md` 为唯一真源

---

**如何从 0 跑通（最短路径）**
> 下述命令名以 `scripts/` 为准；你也可以用 Makefile 包装成 `make pipeline` 等一键命令。

- **Step0：仓库健康检查（开工前必须跑）**
  - 运行 `scripts/00_validate_repo.py`
  - 目标：路径/配置/schema/模板齐全，避免后续“跑到一半才发现缺字段”

- **Step1：CSV → HDF5（带元数据）**
  - 运行 `scripts/01_csv_to_h5.py`
  - 产物：`data/hdf5/<data_snapshot_id>.h5` + `data/snapshots/<data_snapshot_id>/manifest.json`
  - 注意：此处只做“可靠落盘 + 字段齐全”，不做模型训练

- **Step2：去前 50 帧 + 生成 cell_mean/cell_stats**
  - 运行 `scripts/02_build_cell_mean_stats.py`
  - 产物：`data/derived/cell_mean`、`data/derived/cell_stats`（或写回 HDF5 的 derived 组）

- **Step3：全谱 PLSR（选焦距×能量，按 sample_id Group-CV）**
  - 运行 `scripts/03_step1_focus_energy_plsr.py`
  - 输出：Top-K 组合、推荐 `focus_opt` 与 `energy_opt`
  - 报告：`docs/reports/02_step1_focus_energy_plsr.md`（用模板填写）

- **Step4：分通道延时调优（门宽固定 20 µs，多目标）**
  - 运行 `scripts/04_step2_channel_delay_multiobj.py`
  - 输出资产：
    - `configs/acq_config.yaml` 更新 `delay_opt_by_channel`
    - `configs/feature_spec.yaml / loi_registry.yaml`（冻结版本）
  - 输出点云：
    - `artifacts/pointcloud/frame_point_cloud.*`
    - `artifacts/pointcloud/cell_point_cloud.*`
    - `artifacts/pointcloud/outliers/*`
  - 报告：`docs/reports/03_step2_channel_delay_multiobj.md`、`04_pointcloud_screening_summary.md`

- **Step5：Step3 质量门控阈值校准（qa_threshold_set）**
  - 运行 `scripts/06_step3_threshold_calibrate.py`
  - 输出资产：`configs/qa_threshold_set.yaml` + `threshold_set_id` 注册
  - 报告：`docs/reports/05_step3_threshold_calibration.md`

- **Step6：训练/校准/拒识（可选：方案A优先）**
  - 运行 `scripts/07_train_models.py`
  - 输出：`model_id` + 校准产物（Conformal/误差模型等）
  - 报告：`docs/reports/07_model_training_report.md`

- **Step7：影子模式与发布（V3）**
  - 影子模式：`scripts/10_shadow_mode_runner.py` → `docs/reports/08_online_shadow_mode_report.md`
  - 回归门禁：`scripts/09_run_regression_gate.py` → `docs/reports/09_release_regression_report.md`
  - 发布打包：`scripts/08_export_release_bundle.py` → `artifacts/releases/<release_id>/`

---

**配置与资产（V3 必须遵守的“工程纪律”）**
- **必须分离的两类资产**
  - `acq_config`：采集/门控参数最优配置（focus/energy/gate_width/delay_opt_by_channel…）
  - `qa_threshold_set`：质量门控阈值集（sat/snr/continuum/ood/quality_score_map…）
- **必须绑定的追溯字段**
  - `data_snapshot_id`：数据快照
  - `preprocess_hash`：预处理配置hash
  - `feature_spec_hash`：LOI/特征定义hash
  - `splits_id`：防泄漏切分策略
  - `release_id`：发布版本（绑定 model/acq/threshold/preprocess/feature/calibration）
- **任何一个资产更新**
  - 必须：更新注册表 + 生成/更新报告 + 跑回归门禁（golden set）
  - 不允许：只改代码不改配置/不留痕，或只改阈值不做回归

---

**质量门控与输出状态（你在现场会看到的行为）**
- **nominal**：有效帧充足、质量达标、漂移未超限 → 正常输出
- **degraded**：仍输出但明确标注（有效率下降/轻度漂移/OOD_WARN 多）→ 可用于过程监测但需谨慎
- **reject**：数据不足/质量硬失败/OOD_FAIL/漂移严重 → 拒识（避免误导性输出）

---

**文档体系（docs/ 与 docs/reports/）**
- `docs/data_dictionary.md`：HDF5 字段字典（唯一真源）
- `docs/design/interfaces.md`：Raw/Preprocessed/Inference/Fusion 字段与含义
- `docs/protocol_alignment.md`：化验对齐、延迟Δ、混样噪声与防泄漏
- `docs/protocol_evaluation.md`：FAT/SAT/回归门禁与指标口径
- `docs/ops/*`：运维SOP、失败处理矩阵、发布与回滚流程
- `docs/reports/*`：所有实验与发布必须有对应报告（模板已提供，可复制填写）

---

**测试与验收（对外可解释、对内可回归）**
- **单元/一致性测试（tests/）**
  - schema 校验、CSV解析一致性、HDF5 roundtrip、切分防泄漏、门控口径、1Hz融合口径、注册表一致性
- **验收指标（不在 README 里虚构数值）**
  - 误差：MAE/RMSE/Bias/P95（分层）
  - 稳定性：1 Hz 方差/漂移斜率
  - 拒识：Reject Rate、False Reject、False Accept、有效数据率
  - 校准：覆盖率（PICP）与分层可靠度
  - 时延：单谱 P95、1 Hz 输出延迟
  - 漂移：spec_shift/embedding_ood/threshold_drift 趋势与触发记录

---

**协作方式（给新同事/外部读者）**
- **想“理解系统”**：先读 `docs/design/architecture.md` 与 `docs/design/interfaces.md`
- **想“跑通数据”**：按“如何从 0 跑通”的 Step0–Step5
- **想“改一个模块”**：
  - 先查 `docs/data_dictionary.md` 与对应 `configs/*.yaml`
  - 修改后必须更新：hash/注册表/报告/回归门禁
- **想“上线/回滚”**：
  - 只通过 `release_id` 发布；发生问题按 `docs/ops/release_process.md` 回滚

---

**安全与合规声明（强制）**
- 本仓库仅覆盖：光谱数据处理、参数调优、质量门控、模型训练与评估、漂移监测与版本治理。
- 不包含任何激光/高压/联锁/现场危险操作步骤；采集参数调整与现场运维必须由具备资质人员按安全规程执行。
- 本系统输出用于“过程监测与辅助决策”，不得替代安全联锁或高风险工序的安全控制。

---

**致谢与引用（可选）**
- 若需添加参考文献与类似工作对照，请在 `docs/reports/99_appendix_tables.md` 或独立 `docs/references.md` 维护，并给出真实可核验来源（期刊/年份/DOI）。
