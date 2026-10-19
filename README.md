# MicroDuck 学习笔记

MicroDuck 组队学习打卡笔记（Task 0 / Task 1 / Task 2 / Task 3）。

| 笔记 | 内容 |
|---|---|
| [Task 0：MuJoCo 策略控制 HelloWorld](notes/Task0笔记_HelloWorld闭环.md) | 控制闭环入门：速度命令 → 观测 → 策略 → 14 维关节目标 → 物理步进 |
| [Task 1：MicroDuck 篮球平衡](notes/Task1笔记_篮球平衡.md) | 完整任务链路：61 维观测、LSTM blind actor、PPO + privileged critic、ONNX/GPU/BPU 分工 |
| [Task 2：浏览器物理扰动](notes/Task2笔记_浏览器物理扰动.md) | 部署/展示侧：mjswan 把策略打包进浏览器，MuJoCo WASM + ONNX Runtime Web 实时推理，鼠标拖拽写入外力数组的交互验证 |
| [Task 3：高跷行走（实验报告）](notes/Task3笔记_高跷行走.md) | 形态编译 + 逐级课程学习：高跷作为接触几何进入 MuJoCo，公开 PPO 策略（25 cm / 2 m）本地 ONNX 回放与视频渲染 |
| [Task 3：验收笔记（打卡）](notes/Task3验收笔记_高跷行走.md) | 预测→验证记录、学习者口述的认知修正、费曼检验 5 题、亲手完成的错配实验（25cm 策略 + 2cm 形态 → 秒倒）视频证据 |

学习方式约定：[学习方式.md](学习方式.md)（2026-10-20 订立：学习者亲手操作、预测先行、费曼检验，助教不代跑关键实验）。

实验产物（视频与 JSON 报告）在本地仓库 `Microduck/03-Notebook/outputs/`（本机重跑）与 `recordings/`（发布版）。
