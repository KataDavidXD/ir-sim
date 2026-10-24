# 诊断证据与复现说明

> 数据日期：2026-10-24。基线：`39ed2f0e4ca006f5061d8808b19b46b2cfff2fca`。本目录保存分析阶段的诊断材料，不包含 ECS 或 MCP 的产品实现。

## 1. 文件与证据边界

| 文件 | 内容 |
| --- | --- |
| [baseline-benchmarks.json](baseline-benchmarks.json) | 9 个基线场景、25 个原始计时样本、阶段探针、profile、快照编码和 SFM 分配峰值 |
| [baseline-harness.py.txt](baseline-harness.py.txt) | 实际运行的临时诊断脚本；保存为文本，便于审阅并按下面命令在临时目录复现 |
| [lidar-snapshot.json](lidar-snapshot.json) | 从真实运行取得的完整 WorldState 字典，帧 45、4 个机器人、100 个静态障碍物 |
| [gemma-probe.json](gemma-probe.json) | 实际模型请求、响应、时间、HTTP 错误、正确性检查与限制 |
| [gemma-probe.py.txt](gemma-probe.py.txt) | 对现有本地 HTTP 服务执行的临时能力探测；不是 MCP Server |
| [validation.json](validation.json) | 已运行验证的命令、结果与范围 |

JSON 中的计时和模型响应保留实际值。profiler 路径已规范化，去除不必要的本机绝对路径；这不改变计时。没有保存认证凭据或其他 Docker 项目配置。

## 2. 机器与依赖

Windows 11，AMD Ryzen 9 9955HX（16 核／32 逻辑处理器）。诊断使用仓库 uv 环境中的 Python 3.13.13、NumPy 2.3.2、Shapely 2.1.2、SciPy 1.16.1、Matplotlib 3.10.9。`OPENBLAS_NUM_THREADS=1`、`OMP_NUM_THREADS=1`。

记录表中的最终基准在核心 pytest 进程完成、临时 Gemma 容器停止后运行；较早的探索性计时没有用于本表。桌面操作系统仍可能有背景负载、功耗和热状态变化，这不是隔离的性能实验室。

## 3. 基准方法

- 单进程、`headless=True`、seed 42；不调用 render，没有传感器噪声。
- 每个场景预热 5 步，随后记录 25 次未插桩 `step()` 耗时，给出中位数与样本 p95。
- 再继续执行 10 步阶段探针和 5 步 cProfile；这不是前 25 步的精确分解。场景继续演化，接触和传感器候选可能变化。
- `build_tree` 是 `_objects_step`／`_contact_step` 的嵌套成本，不能再次相加。cProfile 累计时间也是嵌套的，且会放大 Python 调用成本。
- 最后重复 7 次快照复制、`to_dict` 和 JSON 编码，分别取中位数；结果不包含模型推理时间。
- 所有场景由 harness 构造并记录 YAML SHA-256；contact 场景直接使用上游 `usage/26push_box_world/push_box_world.yaml`。

当前只有一轮短记录，不足以给出稳定吞吐、可靠尾延迟或跨机器比较。正式性能 PR 应按 ECS 方案扩大重复次数和场景矩阵。

## 4. 实际基线结果

单位：ms。下表是现有实现的耗时，没有任何新 ECS 后端参与。

| 场景 | 对象／感知设置 | step 中位数 | step p95 |
| --- | --- | ---: | ---: |
| `dash_100` | 100 个 diff 圆形机器人 | 3.924 | 4.319 |
| `dash_500` | 500 个 diff 圆形机器人 | 33.857 | 36.252 |
| `dash_1000` | 1000 个 diff 圆形机器人 | 163.749 | 177.907 |
| `prescribed_1000` | 1000 个机器人，显式动作＋1000 个目标 ID | 64.368 | 80.194 |
| `sfm_individual_200` | 200 个逐对象 SFM | 102.574 | 122.752 |
| `sfm_group_200` | 200 个组 SFM | 14.308 | 17.319 |
| `sfm_group_1000` | 1000 个组 SFM | 204.049 | 242.403 |
| `lidar_4_static_100` | 4 个机器人各 360 beams，100 个静态矩形 | 15.791 | 17.311 |
| `contact_push_box` | 上游推箱子示例，3 个机器人、共 7 个对象 | 1.138 | 1.808 |

合成机器人初始排在间距 1.2 m 的网格上，半径 0.15 m，目标为沿 x 方向 200 m 以外。dash／SFM 速度上限 1 m/s，显式动作场景为 `[0.3, 0]`；它们不是严格相同的轨迹任务，不能用两者之比声称消除某段代码后的加速倍数。SFM 个体和组行为的更新观测语义也不同。

本次 1000 个 dash／组 SFM 的每步时间都超过默认 100 ms 步长，说明这些合成负载在这次记录中未达到按墙钟 10 Hz 推进。小规模场景可更快，不能把这个判断扩大为所有 IR-SIM 场景。

### 快照代价

| 场景 | `get_msg` | `to_dict` | JSON 编码 | 完整 JSON 大小 |
| --- | ---: | ---: | ---: | ---: |
| `dash_100` | 0.735 | 1.938 | 0.887 | 96,897 bytes |
| `dash_1000` | 15.799 | 32.334 | 15.191 | 972,773 bytes |
| `sfm_group_1000` | 11.803 | 24.534 | 10.888 | 990,284 bytes |
| `lidar_4_static_100` | 0.948 | 2.392 | 1.744 | 147,884 bytes |

1000 对象的完整 ROS 风格结构接近 1 MB，复制和文本展开已有可见成本；MCP 不宜每次默认返回全部数据。`ObjectState` 同时有首个 `scan` 和 `scans` 列表，完整字典还会包含这些兼容字段。摘要应按场景需要投影，不能把原始字典大小直接当作模型 token 数。

### SFM 内存探针

单独调用已有 `SocialForceModelBatch.social_force()`，使用固定随机位置，记录这段调用内 tracemalloc 可见的新增分配峰值：

| N | 峰值 bytes | 约 MiB |
| --- | ---: | ---: |
| 100 | 1,936,104 | 1.85 |
| 500 | 48,268,936 | 46.03 |
| 1000 | 193,034,936 | 184.09 |

它不是整个进程的 RSS，也不包含所有已存在的对象、几何和消息。增长趋势与代码的密集成对矩阵吻合；未运行 N=10,000，不把外推当作测量。

### profiling 支持的判断

- dash 1000 的 `external_objects` 每步为所有机器人扫描全量对象。5 步 profile 中该路径约一千万次 ID getter 调用，属于可优先消除的结构成本。
- 显式 1000 个目标 ID 时，`_target_positions → position → find_object_by_identity` 反复线性查找，另有 `objects.index`。应单独评估一次性映射。
- LiDAR 场景的 profile 显示边界提取／几何查询占据显著部分；现有 `cast_ray_segments` 本身已经向量化，优化方向应覆盖数据准备。
- 组 SFM 1000 的密集力计算已经是主要阶段之一；只批量运动学不能消除这个成对矩阵问题。

这些是对调用结构和样本的解释，不是新实现的速度保证。

## 5. 本地模型探测

既有 `gemma4-vllm` 服务模型为 `gemma-4-e2b-it`，地址 `http://127.0.0.1:8000/v1`。原容器停止，测试时临时启动，测试后恢复停止；原配置未改。

实际快照的 4 个机器人最小范围依次为 1.05、0.1、1.05、0.1 m。Host 向模型发送该摘要，模型返回帧号 45、数量 4、`robot_1`，属于两个并列最小值机器人之一。请求 197 个输入 token、42 个输出 token，完整响应耗时 19.8968 s；输出包含 JSON 围栏，测试去除单个围栏后校验通过。

`tool_choice=auto` 与指定工具都 HTTP 400，服务说明缺少工具调用解析选项。这个结果针对当前 vLLM 配置，不能据此否定 Gemma 模型的全部工具能力。未验证完整 MCP 链路。详细限制和后续设计见 [MCP 方案](../mcp-gemma-plan.zh-CN.md)。

`gemma-probe.json` 中保留单名 expected fixture，并额外记录两个合法并列答案，避免把 `robot_3` 错误判为失败。该脚本是能力 smoke probe，不是完整模型评测集。

## 6. 复现命令（PowerShell）

在仓库根目录运行。诊断代码先复制到临时目录，生成数据也写到该目录；不修改产品源码。

```powershell
uv sync --locked --group test --group lint
if ($LASTEXITCODE -ne 0) { throw 'Dependency sync failed' }

$reviewScratch = Join-Path $env:TEMP 'ir-sim-review-reproduction'
New-Item -ItemType Directory -Force -Path $reviewScratch | Out-Null
Copy-Item -LiteralPath '.\docs\architecture\evidence\baseline-harness.py.txt' -Destination (Join-Path $reviewScratch 'benchmark.py')
$env:OPENBLAS_NUM_THREADS = '1'
$env:OMP_NUM_THREADS = '1'
.\.venv\Scripts\python.exe (Join-Path $reviewScratch 'benchmark.py')
```

这会生成场景 YAML、`baseline-benchmarks.json` 和 `lidar_snapshot.json`。结果随设备与负载变化，不要求复现逐位相同的耗时；同基线、同依赖和固定 seed 的场景／快照 hash 才是输入核查项。场景 SHA-256 按原始文件字节计算，跨平台换行符差异也会改变它，需要先核对输入内容。

模型探测需本地现有服务可用，可先启动用户自己的 `gemma4-vllm`，检查 `/health` 后再运行：

```powershell
Copy-Item -LiteralPath '.\docs\architecture\evidence\gemma-probe.py.txt' -Destination (Join-Path $reviewScratch 'gemma_probe.py')
.\.venv\Scripts\python.exe (Join-Path $reviewScratch 'gemma_probe.py')
```

探测脚本硬编码本次模型名和 loopback 地址；在其他模型或端口使用前应修改临时副本。它读取上一条命令生成的实际快照，执行摘要问答和工具能力探测，单请求超时 55 s。包含读取模型响应的有界测试，不启动或修改 MCP 服务。

### 已运行的代码验证

```powershell
.\.venv\Scripts\python.exe -m pytest -q --no-cov tests/test_kinematics.py tests/test_env.py tests/test_objects.py tests/test_geometry.py tests/test_sensors.py tests/test_contact.py tests/test_sfm.py tests/test_rvo_vectorized.py tests/test_behaviors.py
.\.venv\Scripts\ruff.exe check
.\.venv\Scripts\ruff.exe format --check
```

结果：733 passed、7 skipped、24 warnings，79.32 s；Ruff check 全部通过，187 个 Python 文件满足现有格式。warning 来自已有 Agg 非交互显示与 ImageIO 接口弃用提示。本次没有为文档新增产品测试，也没有运行剩余 7 个 test 文件；不能声称全仓测试全部通过。

文档检查包含本地链接、Markdown 解析、引用与 fence、JSON 严格解析、测量值／摘要一致性、输入快照 SHA-256 和 `git diff --check`。规划文件不在 Sphinx source 目录，本阶段未执行 Sphinx 站点构建。
