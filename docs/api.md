## OpenEvolve 中文 API 文档

本文档涵盖 OpenEvolve 的公共 API、函数与类，包含用途说明与示例代码。示例默认以 Python 为主，命令行以 Bash 为主。

### 目录

- OpenEvolve 总览与快速开始
- CLI 命令 `openevolve-run`
- 控制器 `openevolve.controller.OpenEvolve`
- 配置模块 `openevolve.config`
- 评测器 `openevolve.evaluator.Evaluator`
- 评测结果 `openevolve.evaluation_result.EvaluationResult`
- 数据库 `openevolve.database.Program` 与 `ProgramDatabase`
- LLM 模块 `openevolve.llm`
- Prompt 模块 `openevolve.prompt`
- 并行控制 `openevolve.process_parallel.ProcessParallelController`
- 迭代（内部）`openevolve.iteration`

---

## OpenEvolve 总览

- 核心类：`OpenEvolve`
- CLI 入口：`openevolve-run`
- 主要流程：采样 → 生成 → 评估 → 存档/并行/检查点 → 输出最佳方案

### 快速开始

```bash
openevolve-run path/to/initial_program.py path/to/evaluation.py \
  -c configs/default.yaml -o ./openevolve_output -i 200 -t 0.95
```

> `evaluation.py` 至少需要导出 `evaluate(program_path)` 函数，可选导出 `evaluate_stage1/2/3` 以启用级联评测。

---

## CLI 命令

- 入口：`openevolve-run`（注册于 `pyproject.toml`）
- 实现：`openevolve.cli:main`

### 用法

```bash
openevolve-run INITIAL_PROGRAM EVALUATION_FILE \
  [-c CONFIG] [-o OUTPUT] [-i ITERATIONS] [-t TARGET_SCORE] \
  [--log-level {DEBUG,INFO,WARNING,ERROR,CRITICAL}] \
  [--checkpoint PATH] \
  [--api-base URL] [--primary-model NAME] [--secondary-model NAME]
```

常用示例：

```bash
openevolve-run app.py evaluate.py -i 300 -t 0.9 \
  --log-level INFO --checkpoint ./openevolve_output/checkpoints/checkpoint_100
```

---

## 控制器 OpenEvolve

- 路径：`openevolve.controller.OpenEvolve`
- 职责：协调演化流程、并行评测、日志与检查点、保存最佳方案

### 初始化

```python
from openevolve import OpenEvolve
from openevolve.config import Config

oe = OpenEvolve(
  initial_program_path="examples/hello.py",
  evaluation_file="examples/eval.py",
  config=Config(),              # 或 config_path="configs/default.yaml"
  output_dir="./openevolve_output"
)
```

参数：

- `initial_program_path: str` 初始程序文件路径
- `evaluation_file: str` 评测脚本路径，需包含 `evaluate`
- `config_path: Optional[str]` YAML 配置路径
- `config: Optional[Config]` 直接传入配置对象（优先级高）
- `output_dir: Optional[str]` 输出目录

### 运行

```python
best_program = await oe.run(
  iterations=500,
  target_score=0.98,
  checkpoint_path=None
)
```

返回：`Program | None`。完成后将最佳方案代码与指标写入 `output_dir/best`。

---

## 配置模块 openevolve.config

### `Config`

- 字段（节选）：
  - `max_iterations: int`, `checkpoint_interval: int`, `log_level: str`, `random_seed: Optional[int]`
  - `diff_based_evolution: bool`, `max_code_length: int`
  - 早停：`early_stopping_patience: Optional[int]`, `convergence_threshold: float`, `early_stopping_metric: str`
  - 组件：`llm: LLMConfig`, `prompt: PromptConfig`, `database: DatabaseConfig`, `evaluator: EvaluatorConfig`

构造与序列化：

```python
from openevolve.config import Config

cfg = Config.from_yaml("configs/default.yaml")
cfg.to_yaml("configs/out.yaml")
```

### `LLMConfig` 与 `LLMModelConfig`

- 支持通过 `models` 与 `evaluator_models` 定义模型集合与权重；兼容 `primary_model(_weight)` 与 `secondary_model(_weight)`。

### `PromptConfig`

- 关键字段：`system_message`, `evaluator_system_message`, `num_top_programs`, `num_diverse_programs`, `include_artifacts`, `template_dir` 等。

### `DatabaseConfig`

- 关键字段：`population_size`, `archive_size`, `num_islands`, `feature_dimensions`, `feature_bins`, `migration_interval`, `migration_rate`，artifact 配置等。

### `EvaluatorConfig`

- 关键字段：`timeout`, `max_retries`, `cascade_evaluation`, `cascade_thresholds`, `parallel_evaluations`, `use_llm_feedback`, `llm_feedback_weight`。

---

## 评测器 Evaluator

- 路径：`openevolve.evaluator.Evaluator`
- 作用：执行 `evaluation.py` 并返回指标，支持级联评测与 LLM 质评、并发评测与 artifacts 记录。

### 初始化

```python
from openevolve.evaluator import Evaluator
from openevolve.config import EvaluatorConfig

ev = Evaluator(
  config=EvaluatorConfig(parallel_evaluations=2, cascade_evaluation=True),
  evaluation_file="examples/evaluation.py",
  llm_ensemble=None,
  prompt_sampler=None,
  database=None,
  suffix=".py"
)
```

### 方法

- `await evaluate_program(program_code: str, program_id: str="") -> Dict[str, float]`
- `await evaluate_multiple(programs: List[Tuple[str, str]]) -> List[Dict[str, float]]`
- `get_pending_artifacts(program_id: str) -> Optional[Dict[str, Union[str, bytes]]]`

### 评测脚本（用户自定义）

```python
# 必须
def evaluate(program_path: str) -> Dict[str, float] | EvaluationResult:
    ...

# 可选多阶段（启用 cascade_evaluation 时生效）
def evaluate_stage1(program_path: str): ...
def evaluate_stage2(program_path: str): ...
def evaluate_stage3(program_path: str): ...
```

建议输出包含 `combined_score`，否则系统回退到对数值指标取平均。

---

## 评测结果 EvaluationResult

- 路径：`openevolve.evaluation_result.EvaluationResult`
- 字段：`metrics: Dict[str, float]`，`artifacts: Dict[str, Union[str, bytes]]`
- 说明：对旧版 `Dict[str, float]` 保持兼容；`artifacts` 支持保存文本和二进制信息。

示例：

```python
from openevolve.evaluation_result import EvaluationResult

return EvaluationResult(
  metrics={"combined_score": 0.91, "accuracy": 0.93, "runtime": 0.45},
  artifacts={"stdout": "...", "plot.png": b"..."}
)
```

---

## 数据库 Program 与 ProgramDatabase

- 路径：`openevolve.database`

### `Program`

字段要点：`id`, `code`, `language`, `metrics`, `parent_id`, `generation`, `iteration_found`, `metadata`, `artifacts_json`, `artifact_dir`。

### `ProgramDatabase`

核心方法（公共接口）：

- `add(program: Program, iteration: int | None = None, target_island: int | None = None) -> str`
- `get(program_id: str) -> Optional[Program]`
- `sample(num_inspirations: Optional[int] = None) -> Tuple[Program, List[Program]]`
- `get_best_program(metric: Optional[str] = None) -> Optional[Program]`
- `get_top_programs(n: int = 10, island_idx: Optional[int] = None) -> List[Program]`
- `save(path: Optional[str] = None, iteration: int = 0) -> None`
- `load(path: str) -> None`
- `set_current_island(island_idx: int) -> None`
- `next_island() -> int`
- `increment_island_generation(island_idx: Optional[int] = None) -> None`
- `should_migrate() -> bool`
- `migrate_programs() -> None`
- `get_island_stats() -> List[dict]`
- `log_island_status() -> None`
- `store_artifacts(program_id: str, artifacts: Dict[str, Union[str, bytes]]) -> None`
- `get_artifacts(program_id: str) -> Dict[str, Union[str, bytes]]`
- `log_prompt(program_id: str, template_key: str, prompt: Dict[str, Any], responses: List[str]) -> None`

使用示例：

```python
from openevolve.database import Program, ProgramDatabase
from openevolve.config import DatabaseConfig

db = ProgramDatabase(DatabaseConfig())
pid = db.add(Program(id="p1", code="print('hi')", metrics={"combined_score": 0.5}))
parent, inspirations = db.sample(num_inspirations=3)
best = db.get_best_program()
db.save("./checkpoints/checkpoint_0")
```

---

## LLM 模块

### `LLMInterface`

- 路径：`openevolve.llm.base.LLMInterface`
- 方法：
  - `async generate(prompt: str, **kwargs) -> str`
  - `async generate_with_context(system_message: str, messages: List[Dict[str, str]], **kwargs) -> str`

### `OpenAILLM`

- 路径：`openevolve.llm.openai.OpenAILLM`
- 说明：对接 OpenAI 兼容接口，支持标准与推理模型参数、重试与超时、种子控制等。

### `LLMEnsemble`

- 路径：`openevolve.llm.ensemble.LLMEnsemble`
- 方法：
  - `async generate(prompt: str, **kwargs) -> str`
  - `async generate_with_context(system_message: str, messages: List[Dict[str, str]], **kwargs) -> str`
  - `async generate_multiple(prompt: str, n: int, **kwargs) -> List[str>`
  - `async parallel_generate(prompts: List[str], **kwargs) -> List[str>`
  - `async generate_all_with_context(system_message: str, messages: List[Dict[str, str]], **kwargs) -> List[str]`

---

## Prompt 模块

### `TemplateManager`

- 路径：`openevolve.prompt.templates.TemplateManager`
- 方法：`get_template(name)`, `get_fragment(name, **kwargs)`, `add_template`, `add_fragment`

### `PromptSampler`

- 路径：`openevolve.prompt.sampler.PromptSampler`
- 作用：组合系统提示、用户提示、历史/最佳/灵感程序、评测 artifacts，生成演化提示。
- 关键方法：
  - `set_templates(system_template: Optional[str] = None, user_template: Optional[str] = None) -> None`
  - `build_prompt(..., template_key: Optional[str] = None, program_artifacts: Optional[Dict[str, Union[str, bytes]]] = None, feature_dimensions: Optional[List[str]] = None, ...) -> Dict[str, str]`

示例：

```python
from openevolve.prompt.sampler import PromptSampler
from openevolve.config import PromptConfig

sampler = PromptSampler(PromptConfig())
prompt = sampler.build_prompt(current_program="print('hi')", language="python")
```

---

## 并行控制 ProcessParallelController

- 路径：`openevolve.process_parallel.ProcessParallelController`
- 说明：进程池并行，支持岛屿隔离与迁移、批次调度、早停与目标分数触发。
- 关键方法：
  - `start() -> None`
  - `stop() -> None`
  - `request_shutdown() -> None`
  - `async run_evolution(start_iteration: int, max_iterations: int, target_score: Optional[float] = None, checkpoint_callback=None)`

---

## 迭代（内部）iteration

- `run_iteration_with_shared_db(...)`：给持久 worker 使用的单次演化执行器，通常由并行控制器调用。

---

## 常见问题

- 未提供 `combined_score`：系统将对数值型指标取平均作为代理用于演化。
- 评测超时：返回 `timeout=True` 并可在 artifacts 中查看上下文。
- 提示模板：可通过 `PromptConfig.template_dir` 指向自定义模板目录覆盖默认模板。

