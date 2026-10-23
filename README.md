# GUI Intent Execution

研究 GUI 模型已有推理中的多操作意图，能否在实时界面变化下继续执行，从而减少重复调用主模型。

**当前阶段：motivation 实验设计。未实现执行器，未运行实验，无效果结论。**

## 核心问题

1. 一轮可见 reasoning 是否明确提出多个操作意图？
2. 当前 action / 原生 batch 实际覆盖多少意图？
3. 未覆盖意图在后续观察中是否仍然成立，或只需少量执行适配？
4. 其中有多少确实能在不作新任务决策的情况下完成？

例如，reasoning 已决定“选中 A1:C1、加下划线、保存”，但当前输出只点击 A1。后续可能只需重新定位和确认，也可能需要新决策；实验区分这两种情况，不预设前者普遍存在。

## 第一轮设置

| 项目 | 设置 |
|---|---|
| 模型 | UI-Mate-9B、UI-Mate-27B、GUI-Owl-1.5-8B-Think |
| Benchmark | OSWorld-Verified，固定代码与环境版本 |
| 任务 | 同一批 30 个任务：Calc / Writer / Impress / 浏览器 / 系统与文件，各 6 个 |
| 规模 | 每模型每任务 1 次，共 90 条 pilot 轨迹 |
| 主设置 | 仅截图输入、GUI 动作、保留原生 reasoning 和 batch |
| 初步标注 | 每模型按任务均衡抽 100 个决策轮次；全轨迹均保存 |
| 后续验证 | 约 20 个可恢复状态，人工辅助完成已有意图，估计可执行潜力 |

不额外提示模型多想几步，不强制拆开 batch，不训练 JEV，不添加新的 planner。公开输出中没有 reasoning 的轮次如实记录。

## 文档

- [MOTIVATION.md](MOTIVATION.md)：实验设置、任务选择、指标与判据。
- [docs/TRAJECTORY_AND_ANNOTATION.md](docs/TRAJECTORY_AND_ANNOTATION.md)：轨迹格式、人工查看报告和标注规范。
- [docs/RUN_HANDOFF.md](docs/RUN_HANDOFF.md)：给实现与运行者的顺序清单。

**这些是实验规范，不是已实现的运行代码。** 不把其他仓库的脚本视为本仓库已有功能。代码实现、具体任务清单和运行结果后续提交。

## 上游来源

- [UI-Mate 代码](https://github.com/Tencent/UI-Mate)
- [UI-Mate-9B](https://huggingface.co/tencent/UI-Mate-9B)
- [UI-Mate-27B](https://huggingface.co/tencent/UI-Mate-27B)
- [GUI-Owl-1.5-8B-Think](https://huggingface.co/mPLUG/GUI-Owl-1.5-8B-Think)
- [MobileAgent / GUI-Owl](https://github.com/X-PLUG/MobileAgent)
- [OSWorld / OSWorld-Verified](https://github.com/xlang-ai/OSWorld)

模型版本、权重 revision、prompt、parser 与软件版本以正式运行前锁定的 manifest 为准。
