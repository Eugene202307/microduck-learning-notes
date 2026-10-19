# Task 3 学习笔记：MicroDuck 高跷行走（课程学习 + ONNX 复现）

> 日期：2026-10-20 ｜ 实验环境：Windows 11，Python 3.12.14，uv 0.12.5，CPU 推理（无 CUDA）
> 实验产物：`notes/recordings/stilts_25cm_rollout.mp4`（4 s）、`notes/recordings/stilts_200cm_rollout.mp4`（10 s）、错配实验 `notes/recordings/mismatch_25cm_policy_on_2cm.mp4`、抽帧图 `frame_25cm.png` / `frame_200cm.png`（均在本仓库）
> 教程：[01-任务资料/03-MicroDuck高跷行走强化学习复现](https://github.com/datawhalechina/every-embodied/tree/main/16-专题组队学习/05-MicroDuck运动策略与任务开发专题/01-任务资料/03-MicroDuck高跷行走强化学习复现)；源码固定版本 `c5fcc50`

---

## 1. 这个实验是干什么的（一句话）

给双足小鸭子装上"高跷"——不是在画面里把腿拉长，而是把高跷作为**新的接触几何、质量和惯性**显式编译进 MuJoCo 机器人描述，再用逐级课程学习训出的公开 PPO 策略（25 cm 和 2 m），在本地做 ONNX 推理并渲染 rollout 视频。

验证的是一条链路：**形态编译 → 公开 checkpoint 回放 → 本地视频**。不是重新训练。

## 2. 我原来以为的 vs 实际上的（认知修正）

| 我原来的想法 | 实际发现 |
|---|---|
| 高跷就是把腿在渲染里拉长 | 不是。`stilt_constants.py` 会计算支撑环/安装环、生成凸形 mesh、在脚踝下新增 `stilt_left/right` body、加承重碰撞 geom + 紫色可视 geom + 新 foot site，同时把旧脚底碰撞改名为 `*_original_sole_disabled` 关闭。物理变了，策略面对的才是新问题 |
| 一个策略能通吃各种高度 | 不能。每个 `policy.onnx` 和对应高度、质量、blend 绑定。25 cm 策略配 25 cm 形态，拿 2 m 几何套 25 cm policy 会直接倒 |
| 公开 checkpoint.pt 可以"断点续训" | 只是 **actor-exact warm start**：actor 权重和 ONNX 精确对应，但 critic 是新建的、optimizer moments 清空、lr 只有 1e-5。manifest 里写得很清楚（`critic: fresh-compatible-scaffold`） |
| 视频是预录动画或关节角度回放 | 不是。策略是 50 Hz 闭环控制器：每 20 ms 读一次 61 维本体感觉观测，输出 14 维关节位置偏移（围绕 HOME 姿态，scale=1.0 rad），视频是策略与物理实时交互的结果 |
| 课程学习是"直接上 2 m" | 是一条 2 cm → 2 m 的连续迁移链：先在宽支撑（blend=0）上学走，再收窄支撑到 blend=0.5（约 17 mm × 22 mm 末端），再逐级升高度。每阶段只让一个变量变难，失败时可归因 |
| 中文路径的坑 Task2 已经踩过，应该记得避开 | 还是踩了（见踩坑记录）。junction 和 8.3 短路径都不行，最终把 `02-可运行代码` 整体改名为 `02-runnable-code`，一劳永逸 |

## 3. 运行链路（我理解的流程）

```
形态编译（每次启动环境时）
─────────────────────
MICRODUCK_STILT_HEIGHT_CM + MICRODUCK_STILT_BLEND
        │
        ▼
stilt_profile_dimensions → stilt_mesh_data → MjSpec mesh 资产
        │
        ▼
ankle 下新增 stilt body（碰撞 geom + 可视 geom + foot site）
旧 sole collision 改名关闭 → ManagerBasedRlEnv

部署推理（每 20 ms 一步）
─────────────────────
61 维观测（角速度 3 + 投影重力 3 + 关节位置 14 + 关节速度 14
          + 上一步动作 14 + 速度命令 3 + 头/身状态 10）
        │
        ▼
ONNX actor（归一化已烘焙进模型）→ 14 维关节位置偏移
        │
        ▼
执行器 + MuJoCo Warp 接触动力学 → 下一步观测
```

**课程链**（公开训练记录，见教程 §4.2）：400 里程碑 2 cm/blend 0 → 600 收窄到 blend 0.5 → 2000 巩固 5 cm → 2800 到 25 cm（本次用的 checkpoint）→ 一路升到 6500 里程碑的 2.0 m。manifest 里 25 cm 对应 `checkpoint_iteration: 2800`，和课程表对得上。

**质量模型**：每根高跷 `0.012 kg + 0.001 kg/cm × 高度`。25 cm → 37 g/根，2 m → 212 g/根。这是仿真参数，不是打印件称重。

## 4. 实验记录（证据）

### Checkpoint 1：环境与任务注册 ✅
- `uv sync` 完成（torch、warp-lang 1.12.0、mjlab 等）
- `uv run list-envs` 中出现 `Mjlab-Stilt-Flat-MicroDuck`（编号 33）

### Checkpoint 2：下载公开策略 ✅
- `hf download HannesVonEssen/microduck-stilts`：25cm 和 200cm 的 `policy.onnx`（793,759 B）、`checkpoint.pt`（1,617,077 B）、`manifest.json`
- 25cm `policy.onnx` 的 sha256 前缀 `04eb9e3cf42f08fc` 与 manifest 中 `policy_onnx_sha256` 一致

### Checkpoint 3：ONNX 冒烟测试 ✅
- CPU InferenceSession：`obs [1, 61] → actions [1, 14]`，与教程和 manifest（`obs_len: 61, action_len: 14, action_scale: 1.0`）一致

### Checkpoint 4：25 cm rollout 视频 ✅
- `scripts/export.py`，`HEIGHT_CM=25, BLEND=0.5`，CPU，200 步
- 产物：`stilts_25cm_rollout.mp4`，实测 4.0 s、50 FPS、1280×720
- 抽帧（第 100 帧）：鸭子直立，紫色高跷清晰可见，双高跷末端落地承重，无倒地

### Checkpoint 5：2 m rollout 视频 ✅
- `HEIGHT_CM=200, BLEND=0.5`，500 步
- 产物：`stilts_200cm_rollout.mp4`，实测 10.0 s、50 FPS、1280×720
- 抽帧（第 250 帧）：相机明显拉远，鸭子本体变小，2 m 高跷直立承重——和教程描述的"摄像机距离随高度增大"一致
- 两份导出末尾均打印 `Written artifacts/stilts_*.onnx`

## 5. 踩坑记录（这次真实的坑）

1. **中文路径让 MuJoCo 打不开 XML（Task2 的坑重现）**：第一次 `export.py` 报 `ParseXML: Error opening file .../mjlab/scene/scene.xml`，文件明明存在且内容正常。最小复现确认：相对路径能开、含 `02-可运行代码` 的绝对路径打不开——MuJoCo 的 C 层 `fopen` 处理不了非 ASCII 路径。
2. **目录联接（junction）没用**：建了 `run-stilts` 联接指向项目，但 Python 启动时会把联接解析成真实路径，`sys.executable` 和模块 `__file__` 又变回中文路径。
3. **8.3 短路径没启用**：`%~sI` 返回的还是长文件名，E 盘没开 8.3 命名。
4. **临时解法（沿用 Task2 思路）**：先用 `robocopy` 把项目（排除 `.venv`）复制到纯 ASCII 目录 `stilts-run\`，完成了当天的两段渲染。
5. **最终解法：直接把目录改名为英文**。当天晚些时候把整个 `02-可运行代码` 迁移为 `02-runnable-code`：迁移全部子目录 → 批量替换 30 个文件中的 113 处旧路径引用（Notebook 的向上查找逻辑、文档链接都在内）→ 重建 `.venv` → 在新路径下重渲 25 cm 视频验证通过。`stilts-run` 副本已删除，`code_playground` 链接已指向新路径。从此工作区代码路径为纯 ASCII，这个坑不会再来第三次。
6. **改名时 `.venv` 不能跟着搬走**：目录迁移后旧 `.venv` 里 uv trampoline 烘焙的还是中文路径，报 `uv trampoline failed to canonicalize script path`。必须删掉 `.venv` 再 `uv sync`（走全局缓存，秒级完成）。
7. **改名遇到的文件锁**：直接 `ren`/`Rename-Item` 根目录被拒（有程序持有目录句柄，子目录一度全部被锁，稍后自动释放）。改用"新建英文目录 + `Move-Item` 逐个迁移子目录（同卷改名，瞬时完成）+ 删除空壳"绕开。
8. **CPU + Windows 可行但只做回放**：MuJoCo Warp 在 CPU 上编译一次后即可跑，200/500 步渲染都在几分钟内完成；完整 PPO 训练（4096 并行环境）还是得 Linux + CUDA。

## 6. 边界声明：这次证明了什么、没证明什么

**证明了：**
- 固定版本源码 + 公开 checkpoint 在本地 Windows/CPU 上可以完整复现 25 cm 和 2 m 高跷行走视频；
- ONNX 接口契约（61 → 14）与 manifest 一致，模型文件校验通过；
- 高跷是真实进入物理仿真的（紫色 mesh、末端承重、相机随高度拉远都能在产物中直接观察）。

**没证明：**
- 没有重新训练任何策略，全部是公开 actor 的闭环回放；
- 2 m 结果是 **simulation-only**（manifest 原话），不能当作打印尺寸或真机方案——真机需要重新校核承载路径、舵机扭矩、跌倒保护；
- checkpoint 续训是 warm start，不等于拿到作者完整训练状态；
- 策略没有视觉，不会识别球、门或障碍，也不能外推到没训练过的高度。

## 7. 下一步可选方向

- 训练 smoke test：`HEIGHT_CM=2 BLEND=0`、64 环境、5 轮迭代，验证训练入口（建议在 Linux + CUDA 上）；
- 从 25 cm checkpoint warm start 续训 100 轮，观察 value loss 和终止率；
- 接 `03-Notebook/03_高跷行走_课程与ONNX_BPU.ipynb` 的 BPU 单元（需要 RDK 开发板）。
