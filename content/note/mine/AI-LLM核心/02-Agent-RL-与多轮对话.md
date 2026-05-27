# Agent RL 与多轮对话强化学习

**相关理论文档**：[RL 算法对比](../bd/Tech/Tech--RL 算法对比.md)、[强化学习方法](../bd/Tech/Tech--强化学习.md)
**相关实践文档**：[回复模型强化学习](../bd/work/回复模型强化学习.md)、[多轮对话 RL](../bd/work/多轮对话 RL.md)

这个领域在 2024-2026 年快速迭代，从早期的 Agent 探索进化到系统化的 RL 框架。

## 核心问题与演进方向

### Agent 需要什么
- **长序列规划**：在多步交互中做出最优决策
- **探索与利用平衡**：不能只跟随已有数据
- **动态反馈**：实时调整策略而非离线学习

### 从 Agent 到 RL 的转变
- **早期**：Agent 主要是 "prompt + tool use" 的组合
- **当前**：引入强化学习来优化**路径选择**和**奖励信号**

## 重要框架与技术

### 基础框架
| 框架 | 特点 | 适用场景 |
|------|------|---------|
| **veRL** | 分布式 RL 框架 | 大规模模型训练 |
| **ROLL** | 在线 RL | 实时反馈场景 |
| **SkyRL** | 高效计算 | 资源受限环保 |

### 关键论文梳理

**2024-2025 Agent RL 热点**

1. **RAGEN / RAGEN-2**（检索增强 Agent RL）
   - 核心：将检索融入 Agent 的决策过程
   - 特色：Turn-level rewards（每轮对话级别的奖励）
   - Repo: https://github.com/RAGEN-AI/RAGEN

2. **Agent-R1**（推理优化的 Agent）
   - 吸收了 OpenAI o1 的思想
   - 强化学习中的长链条推理
   - Repo: https://github.com/AgentR1/Agent-R1

3. **AgentRL / AgentGym-RL**（系统化 RL 训练）
   - 清华开源的系统框架
   - 包含多环境基准（ALFWorld、WebShop 等）
   - Repo: https://github.com/THUDM/AgentRL

4. **SkillRL**（经验蒸馏的进化学习）
   - 创新点：不存储原始轨迹，而是提炼为"技能"
   - 成功轨迹 → 演示性技能
   - 失败轨迹 → 反事实知识（包含失败原因和正确做法）
   - 实现 10-20 倍的 token 压缩

5. **多模态 Agent**（视觉 + 推理）
   - Persona Simulator RL：用模拟用户进行交互训练
   - SimulatorArena：多 Agent 竞争环境

### 经验与教训

**GRPO（Group Relative Policy Optimization）的性能**
- 是当前多轮对话 RL 的标杆算法
- 相比纯 SFT：性能提升 10-30%
- 最耗时的阶段：**rollout 采样**（生成轨迹）和**价值网络训练**

**DPO 与 DAPO 的对比**
- **DPO**：直接偏好优化，简单但有局限
- **DAPO**（Dynamic APO）的创新：
  - `clip-higher`：动态裁剪机制
  - `Dynamic Sampling`：根据实时困难度调整采样策略
  - Token 级别优势计算（不仅是序列级）

**工程踩坑**
- veRL 框架为什么需要重新 forward 计算 log_probs？
  - 答：因为在 rollout 后需要重新评估策略的概率分布
  - 解决方案：使用 kv_cache 加速重复计算

**推理优化 ≠ 训练优化**
- 很多候选生说"用 DeepSpeed 优化"，但 RAGEN/Agent-RL 主要优化的是**推理时**的采样效率
- 真正的瓶颈：采样（rollout）> 前向传播 >> 反向传播

## 多轮对话评估的困境

### 奖励信号的问题
- 自动评估不可靠（LLM 评分偏差大）
- 人工标注成本高且不一致
- **解决思路**：
  - 使用 LLM-as-judge + 多数投票降低单一模型偏差
  - Turn-level rewards 代替 trajectory-level rewards（更细粒度反馈）
  - Process-based 奖励（评价推理过程，不仅是答案）

### Agent 在长序列任务上的挑战
- **WebShop 任务**：涉及多步决策和状态管理
- SkillRL 在 WebShop 达到 72.7% 成功率（相比 GRPO 的 ~50%）
- 关键在于技能库的递归进化机制

## 当前趋势与未来方向

### 短期（2026 年预期）
- 多模态 Agent RL 普遍化（VL-LLM + 在线 RL）
- 开源框架成熟：veRL、AgentRL 等框架的广泛应用
- 混合 RL：DPO + PPO / GRPO 的最佳组合

### 长期方向
- **递归学习**：Agent 自己生成训练数据（如 SkillRL 的技能库动态扩展）
- **跨任务泛化**：通用技能库的构建
- **分布式推理**：多 Agent 协作完成复杂任务

## 关键资源速查

### 论文
```
检索增强Agent: RAGEN, MUA-RL, UGST, SAGE
推理优化: Agent-R1, τ2-bench, AgentGym-RL
经验蒸馏: SkillRL, Goal Alignment
环境基准: ALFWorld, WebShop, ColBench
```

### 开源框架对比
- **veRL**：最完整的分布式框架
- **AgentRL**：最系统的基准+框架结合
- **ROLL**：在线 RL 的轻量级选项

### 面试重点
1. **GRPO vs DPO**：各自的适用场景
2. **Rollout 采样**：为什么是 Agent RL 的瓶颈
3. **技能库设计**：如何权衡知识压缩和召回效率
4. **奖励设计**：多轮对话中的 reward hacking 防护

