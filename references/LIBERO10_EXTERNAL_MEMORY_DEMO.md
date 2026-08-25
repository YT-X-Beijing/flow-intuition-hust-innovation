# 外置记忆增强机器人策略：LIBERO-10 一周 Demo 实验设计

## 1. 实验目标

在不修改基座模型权重、不进行梯度更新的条件下，验证以下外置记忆闭环能否稳定运行：

```text
历史轨迹产生 → 状态—动作片段入库 → 相似阶段检索
→ 动作块门控干预 → LIBERO执行 → 结果回写
```

本实验不追求 SOTA，也不将一次演示解释为完整持续学习。最低目标是链路可运行、记忆可开关、结果可解释，并获得正向、中性或负向的可信结论。

## 2. 固定基座与环境

### 主基座

- 模型：`moojink/openvla-7b-oft-finetuned-libero-10`
- 本地路径：`/root/private_data/download_model/openvla-7b-oft-finetuned-libero-10`
- 输出：连续动作块，形状 `[8, 7]`
- 输入：主视角、腕部视角、8维 proprio、语言指令
- 权重：全程冻结
- 环境：LIBERO-10

### 备用基座

- 模型：`Ericwen2001/pi05_libero_10`
- 本地路径：`/root/private_data/download_model/pi05_libero_10`
- 仅在 OpenVLA-OFT 无法稳定闭环时启用，不在本周同时开发双基座干预逻辑。

## 3. LIBERO-10 任务

| ID | 指令摘要 |
|---:|---|
| 0 | 字母汤和番茄酱放入篮子 |
| 1 | 奶油奶酪盒和黄油放入篮子 |
| 2 | 打开炉灶并放置摩卡壶 |
| 3 | 黑碗放入底层抽屉并关闭抽屉 |
| 4 | 两只杯子分别放到左右盘子 |
| 5 | 书放入置物篮后侧隔间 |
| 6 | 白杯放盘子，巧克力布丁放盘子右侧 |
| 7 | 字母汤和奶油奶酪盒放入篮子 |
| 8 | 两个摩卡壶放到炉灶 |
| 9 | 黄白杯放入微波炉并关闭微波炉 |

## 4. 一周任务范围

核心持续经验序列：

```text
Task 0 → Task 7 → Task 1
```

该序列同时包含：

- 相同目标容器；
- 重叠操作物体；
- 双物体长程操作；
- 新记忆对旧任务的检索干扰。

附加跨任务序列：

```text
Task 2 → Task 8
```

用于观察摩卡壶抓取、运输和炉灶放置子技能的迁移。

## 5. 无数据泄漏协议

每个任务的 initial states 在实验开始前固定划分：

```text
Memory states:     用于生成记忆
Validation states: 用于调整 Top-K、阈值和 alpha
Test states:       只用于最终评测
```

约束：

1. Test state 的轨迹在最终评测前不得进入记忆库。
2. 调参不得使用 Test state 的成功率或视频。
3. 最终评测期间记忆库冻结，Test 轨迹不回写。
4. A/B 使用相同 task、initial state、seed 和 episode 顺序。
5. 当前 episode 不得检索自身或未来轨迹。
6. Oracle 结果只作为诊断上界，不计入正式系统性能。

必须记录：

```text
current_task_id
current_initial_state_id
retrieved_task_id
retrieved_initial_state_id
retrieved_episode_id
retrieved_step_id
similarity
memory_split
alpha
```

## 6. 记忆结构

不以完整 episode 作为单条向量记忆，而按模型重规划时刻存储局部状态—动作片段：

```python
MemoryItem:
    task_id
    task_prompt
    episode_id
    initial_state_id
    step_id
    progress
    visual_embedding
    proprio_state
    gripper_state
    action_chunk       # [8, 7]
    episode_success
    subskill           # 可选规则标签
    split              # memory / validation
```

完整轨迹、视频和环境状态单独存盘，不直接放入向量索引。

本周仅允许成功 episode 的片段贡献动作干预。失败轨迹可以保存并分析，但不参与动作融合。

## 7. 检索设计

### 同任务检索

优先检索相同 `task_id` 的其他 initial states：

\[
S = 0.60S_{visual} + 0.25S_{proprio} + 0.15S_{progress}
\]

推荐初值：

```yaml
top_k: 3
similarity_threshold: 0.82
max_memory_items: 5000
```

### 跨任务检索

只有同任务无高置信记忆时，才检索其他任务的兼容子技能。跨任务候选必须满足：

- 操作平台和动作空间相同；
- gripper 状态兼容；
- 当前子技能兼容；
- 操作物体或目标区域具有可迁移关系；
- 使用更高相似度阈值。

跨任务只迁移局部子技能，不复放完整长程轨迹。

## 8. 动作干预

基座动作与最佳记忆动作均为 `[8, 7]`：

\[
A_{final}=(1-\alpha)A_{base}+\alpha A_{memory}
\]

门控逻辑：

```python
if no_memory:
    alpha = 0
elif similarity < threshold:
    alpha = 0
elif neighbor_action_disagreement > disagreement_threshold:
    alpha = 0
else:
    alpha = alpha_max * confidence
```

安全约束：

- 仅融合前6个机械臂维度；
- gripper 使用基座动作；
- 动作裁剪到环境合法范围；
- 限制 `A_final - A_base` 的最大偏移；
- 每次重规划重新检索，不持续沿用旧片段。

推荐初值：

```yaml
same_task_alpha_max: 0.20
cross_task_alpha_max: 0.08
```

验证阶段仅扫描 `0.10 / 0.20 / 0.30` 三档，不做全网格搜索。

## 9. 实验组

### A：Baseline

- 关闭记忆；
- `alpha = 0`；
- 原生 OpenVLA-OFT。

### B1：同任务跨实例记忆

- 记忆来自相同任务的 Memory states；
- Test states 从未进入记忆库；
- 主实验。

### B2：跨任务子技能记忆

- 目标任务轨迹不进入记忆库；
- 仅允许从先前任务检索兼容子技能；
- 使用更小 alpha。

### Oracle

- 人工指定正确历史轨迹和相近阶段；
- 用于判断动作融合机制是否存在有效上界；
- 不作为正式记忆系统结果。

### Negative（可选）

- 人工提供不兼容阶段记忆；
- 少量运行，用于确认错误检索确实会造成负迁移。

## 10. 评测协议

### 任务筛选

LIBERO-10 十个任务先各运行2次，记录稳定性和基线表现。核心序列仍优先使用 `0 → 7 → 1`；若某任务当前基座完全无法运行，再记录原因并替换。

### 正式配对评测

每个核心任务建议：

- 10个冻结 Test initial states；
- A 与 B1 各10 episodes；
- 相同状态一一配对；
- 时间允许时增加第二个 seed。

跨任务 B2 只选择1个目标任务做 leave-one-task-out 演示。

## 11. 指标

主指标：

- 成功率；
- `rescued episodes`：A失败、B成功；
- `harmed episodes`：A成功、B失败；
- 净救援数：`rescued - harmed`；
- 随经验加入的 prequential 成功曲线。

长程辅助指标：

- 完成的子目标/阶段数；
- 首次失败阶段；
- episode 长度；
- 记忆覆盖率；
- 检索接受率；
- 平均相似度和 alpha；
- 动作偏移量；
- 额外推理延迟；
- 错误检索和负迁移类型。

## 12. 持续经验实验

在 `0 → 7 → 1` 序列中：

```text
阶段0：空记忆库
阶段1：加入 Task 0 的成功记忆
阶段2：加入 Task 7 的成功记忆
阶段3：加入 Task 1 的成功记忆
```

每阶段重新测试已出现任务，但测试轨迹不回写。分析：

- Forward Transfer；
- Retention；
- Negative Transfer；
- Retrieval Interference。

由于基座权重冻结，此处的性能下降不是参数灾难性遗忘，而是外置记忆检索污染或干预错误。

## 13. 一周排期

| 日期 | 工作与交付物 |
|---|---|
| Day 1 | 验证 LIBERO-10 模型闭环；十任务各2次预跑；固定任务与状态划分；基线结果 |
| Day 2 | 轨迹记录、片段化、embedding 提取；离线检索可视化 |
| Day 3 | 动作融合、置信门控、安全裁剪、无记忆严格回退 |
| Day 4 | Oracle 测试；定位阶段对齐、动作坐标与时序问题 |
| Day 5 | 冻结记忆库与超参；运行 A/B1 配对主实验 |
| Day 6 | 运行一个 B2 跨任务实验；制作 rescued/harmed 视频 |
| Day 7 | 固化脚本、配置、结果表和 Demo 演示流程 |

## 14. Go / No-Go 标准

Day 3 必须满足：

- 基座完整 episode 可运行；
- 轨迹可入库并检索；
- 记忆可开关；
- 无记忆时动作与基线严格一致；
- 日志可追溯到具体历史片段。

Day 4 判断：

- Oracle 无效：停止优化检索，排查动作时序、归一化和阶段对齐；
- Oracle 有效、B1 无效：集中优化 embedding 和检索；
- Oracle 与 B1 均有效：进入正式配对实验。

## 15. Demo 成功标准

最低标准：

- A/B/Oracle 可一键切换；
- 至少3个 LIBERO-10 长程任务可运行；
- 外置记忆闭环完整；
- 无测试泄漏；
- 至少展示一个 rescued、harmed 或推进到更晚阶段的案例。

理想标准：

\[
N_{rescued}-N_{harmed}>0
\]

即使最终成功率没有提升，只要闭环、因果顺序和失败分析可信，本 Demo 仍形成有效实验结论。
