# 轨迹保存与多意图标注

## 1. 原始记录必须可以直接人工查看

推荐目录（结构是待实现规范）：

```text
results/<run_id>/<model>/<task_id>/<attempt_id>/
  meta.json
  trajectory.jsonl
  outcome.json
  report.html
  steps/000/
    before.png
    after.png
    request.json
    raw_response.json
    raw_output.txt
    reasoning.txt
    generated_actions.json
    execution_events.jsonl
  annotations/intents.jsonl
  annotations/alignment.jsonl
```

原始日志不可被抽取器、标注和归一化结果覆盖。视频可作为补充，不能替代前后截图。

meta.json 保存：精确 model_id/revision、benchmark commit、task 配置哈希、模型适配器 commit、prompt 和 parser 内容/哈希、解码配置、可见历史管理、分辨率与坐标映射、VM 镜像、GPU/TP/精度、服务软件版本、随机 seed、所有预算与时间戳。request/response 中不存密钥和鉴权头。

每行 trajectory.jsonl 对应一次决策调用，不是一个基础动作。示意：

```json
{
  "run_id": "pilot",
  "model_id": "tencent/UI-Mate-27B",
  "task_id": "<actual_task_id>",
  "attempt_id": "001",
  "decision_index": 0,
  "before": "steps/000/before.png",
  "after": "steps/000/after.png",
  "request": "steps/000/request.json",
  "raw_response": "steps/000/raw_response.json",
  "raw_output": "steps/000/raw_output.txt",
  "reasoning": "steps/000/reasoning.txt",
  "reasoning_status": "visible",
  "generated_actions": "steps/000/generated_actions.json",
  "execution_events": "steps/000/execution_events.jsonl",
  "batch_size_generated": 3,
  "batch_size_submitted": 2,
  "stop_reason": "runtime_guard",
  "input_tokens": null,
  "output_tokens": null,
  "reasoning_tokens": null,
  "request_latency_ms": null,
  "server_generation_ms": null,
  "environment_ms": null
}
```

上例仅展示格式，不是实验数据。未知值用 null。失败状态、空输出、解析失败也记录，不能悄悄丢掉。

reasoning_status 取 visible / absent / truncated / parse_error；区分缺失与解析失败。保留 API 的 content、reasoning_content 等原始字段；不要因启用 reasoning parser 导致正文或推理字段丢失。reasoning 必须来自这次响应。

execution_events 逐动作保存：decision_index、action_index、raw_action、归一化动作、实际坐标与文本、submitted、返回状态、起止时间、异常、截断原因。语义 target 是派生标注，不能覆盖原始坐标。
提交成功不等于意图达成；task_success 放在 outcome.json 由最终 evaluator 给出，逐步局部结果另列。

保留原生 batch 的观察边界。若为了审计逐动作采集额外截图：
- 不向主模型或执行策略额外反馈；
- 固定采集时序并记录审计开销；
- 不能在已有动画/加载时间中假装没有观察开销；
- 无中间截图时，对中间结果可标 unknown，不凭最终截图编造逐动作成功。

## 2. report.html

每条轨迹生成离线可打开的报告，使用相对路径、转义模型输出。
按轮展示：任务与结果、before、完整 reasoning（可展开）、全部生成动作、实际提交/跳过动作、after、耗时/token/异常。
提供相邻 reasoning 对照；可选显示逐动作审计截图和原始响应。
人工必须能分辨：模型没生成后续动作，还是 runner 没执行输出的后续动作。

大图片、视频和完整实验日志保存在实验存储中，不直接提交 Git。仓库只提交去敏案例、任务清单、配置、分析脚本及汇总；记录原始数据位置和校验值，不提交凭据。

## 3. 标注分三遍，防止未来信息泄漏

### 第一遍：意图提取
只看任务、当前/过去观察、当前可见 reasoning；不看当前动作输出、未来截图/动作、任务成败。
每个意图必须引用 reasoning 原文 span，记录：

- intent_id、source_span、操作对象、期望状态、先后顺序；
- status：committed（明确决定）/ conditional（条件分支）/ alternative（备选）/ vague（过于笼统）；
- 当前是否依赖尚未获得的信息；必要前置条件；
- uncertainty 与注释。

只用 committed 计算核心多意图率。不可从任务本身补出 reasoning 没有表达的步骤；不把反思过去、否定动作和代码草稿都当未来计划。
选择 A1、扩选 C1、应用格式可分别标为意图，也可将前两者合为“选择 A1:C1”，但全研究使用统一粒度并保存细粒度子动作映射，避免通过拆分抬高多意图率。

### 第二遍：当前覆盖
再揭示模型输出和当前轮执行记录。
对每个 committed 意图分别标 generated_coverage=full/partial/none/unclear，以及 execution_state=achieved/partial/not_achieved/unknown。
关联 generated action indices、submitted action indices；区分未生成、被 parser 丢失、runtime 截断、动作报错、动作发出但效果不符。
parser 意外丢弃 batch 属实现缺陷，修复后重跑；不能包装为模型本身覆盖不足。

### 第三遍：后续对应
再揭示最多后续 5 个决策周期。
记录意图是否保持、实际对应动作、必要新增信息、局部前置/纠错操作，以及 A/B/C/D 类别：
A 直接继续；B 局部适配但意图保持；C 新决策/计划改变；D 不确定或删失。
“后续意图相同”与“无需再次主模型推理”必须分别判断；后者只能作为待干预验证的候选标签。

## 4. 标注质量

先共同标注少量样本统一粒度，再独立标注。建议对每模型样本的 20% 双人复核，覆盖不同应用及成功/失败轨迹；报告分类一致率及分歧裁决过程。
自动解析器若用于扩大分析，冻结提取规则/模型版本，与盲标人工子集比较 precision/recall；报告无法解析比例。
自动分析器与人工第一遍都不能获得未来信息。

## 5. 统计输出

按模型、任务类别、成功/失败、单动作/多动作输出分组；同一任务的轮次相关，误差估计以任务为单位。
同一 batch 内大量按键或重复点击不能充当多个跨轮 reasoning 复用事件。
保留 uncertain 与 censored 分母，汇总中公开样本流转数量和全部排除原因。
