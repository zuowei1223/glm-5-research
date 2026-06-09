# GLM-5-Turbo 技术参数研究报告

## 研究概述

基于 Hermes Agent Issue #9344、MemOS PR #1310 以及 GitHub 上的多个相关项目，整理了智谱 AI GLM-5-Turbo 模型的关键技术参数和已知问题。

---

## 1. GLM-5 最大输出 Token 限制（max_tokens 上限）

### 官方限制

根据多个项目的实践和测试：

- **默认 max_tokens**: 约 16,000 tokens (根据 Hermes Issue #9344 的 API 验证)
- **有效上下文窗口**: 32,000 tokens (根据 Hermes 用户的 workaround 配置)
- **理论上下文窗口**: 128,000 tokens (但实际有效上下文较小)

### 实测数据（来自 Hermes Issue #9344）

```
输入 tokens: ~53,000 (累积对话上下文)
max_tokens: 默认 (~16K)
结果: reasoning_tokens ~ 16,000, completion_tokens = 16,000, content = ""
finish_reason: "length"

max_tokens: 50 (压力测试)
结果: reasoning_tokens = 50, completion_tokens = 50, content = ""
finish_reason: "length" (100% 推理，0% 内容)
```

### MemOS PR #1310 测试数据

| 模型 | max_tokens | reasoning_tokens | 实际输出 | 状态 |
|------|-----------|-----------------|---------|------|
| glm-5 | 200 | 187 | 空（截断） | ❌ |
| glm-4.7 | 200 | 199 | 空（截断） | ❌ |
| glm-4.7 | 10 | 10 | 空（截断） | ❌ |

---

## 2. 智谱 API 的 finish_reason/截断信号格式

### 标准响应格式

智谱 API 返回的响应结构（OpenAI 兼容）：

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "",  // 可能为空
        "reasoning_content": "模型的思考内容..."  // GLM-5 特有字段
      },
      "finish_reason": "length"  // 截断时的信号
    }
  ],
  "usage": {
    "prompt_tokens": 53000,
    "completion_tokens": 16000,  // 包含 reasoning_tokens
    "total_tokens": 69000,
    "reasoning_tokens": 16000  // 推理消耗的 tokens
  }
}
```

### finish_reason 取值

- **"length"**: 达到 max_tokens 限制，输出被截断
- **"stop"**: 正常结束
- **其他**: 模型特定的停止原因

### 关键问题

**reasoning_tokens 计入 max_tokens**：这是 GLM-5 最大的不同之处。推理 tokens 会消耗整个 max_tokens 预算，导致实际输出内容为空。

---

## 3. GLM-5 在 Hermes Agent 中的已知截断问题

### Issue #9344: Thinking model (glm-5-turbo) reasoning tokens exhaust output budget

**问题描述**：
- 使用 `glm-5-turbo` 作为唯一 provider 时，Hermes 在对话中途返回字面量 `(empty)` 响应
- 根本原因：**reasoning tokens 消耗了全部输出 token 预算**，导致 `content` 为空字符串
- 这是**确定性和渐进性**的故障：随着对话上下文增长，模型在推理上消耗更多 tokens，最终没有空间留给实际响应

**复现步骤**：
1. 配置 Hermes 使用 `glm-5-turbo` 作为唯一 provider（ZAI endpoint: `https://open.bigmodel.cn/api/coding/paas/v4`）
2. 使用 gateway 模式（Discord，但可能影响所有平台）
3. 进行多轮对话，累积工具调用结果（29 个 MCP tool schemas 每次调用注入 ~14.5K tokens 开销）
4. 几轮之后，模型返回 `reasoning_content` 但 `content=""` 且 `finish_reason="length"`

**Hermes 内置恢复机制失败的原因**：

1. **Prefill 恢复适得其反**：当 `reasoning_content` 存在但 `content` 为空时，Hermes 将仅思考的消息添加到对话历史并重试。这会导致上下文增长 → 推理消耗更多 tokens → content 仍然为空。每次重试都会加剧问题。

2. **`_last_content_with_tools` 回退**：如果工具调用先于空响应，Hermes 使用工具调用的中间文本作为最终响应并立即中断 — 完全绕过重试机制。

3. **压缩太迟**：压缩在 API 响应**之后**触发。为时已晚。使用默认的 context_length 猜测 128K（Hermes 无法检测 glm-5-turbo 的实际限制），压缩阈值为 64K — 远高于模型已经失败的 ~53K 上下文。

**临时解决方案**（Workaround）：

```yaml
# config.yaml
model:
  context_length: 32000  # glm-5-turbo 的真实有效上下文
compression:
  threshold: 0.4         # 在 12,800 tokens 时触发压缩
```

### 与 OpenClaw 的对比

| 机制 | Hermes | OpenClaw |
|------|--------|----------|
| 空响应处理 | 向用户发送字面量 `(empty)` | 推理抑制 — 在分发层静默丢弃仅思考的负载 |
| 上下文预算管理 | 响应式：在 API 响应后压缩 | 主动式：`assemble(tokenBudget)` 在 API 调用前强制执行预算 |
| 空响应上的 prefill 恢复 | 添加思考消息 + 重试（增长上下文，更糟） | 短路返回 `{ ok: true }`，不发送内容 |
| 上下文溢出错误处理 | 使用相同上下文重试 | 分类为特殊错误 → 触发压缩/缩减 |
| 工具 schema 开销 | 每次调用发送所有 schema（29 tools = ~14.5K tokens） | 引导文件在 20K 字符处截断，延迟 schema 加载 |

---

## 4. 智谱 API 与其他模型 API 的 max_tokens 行为差异

### OpenAI API
- **max_tokens**: 仅限制输出内容的 token 数量
- **推理 tokens**: 不计入 max_tokens（OpenAI o1 系列除外，有单独的 reasoning_tokens 字段）
- **finish_reason="length"**: 纯粹表示输出内容达到限制

### Anthropic API
- **max_tokens**: 仅限制输出内容
- **无推理 tokens 计入问题**: 模型推理不消耗输出预算
- **行为可预测**: 达到限制时内容部分截断

### 智谱 API（GLM-5）
- **max_tokens**: 限制包括推理 tokens + 输出 tokens 的**总和**
- **reasoning_tokens 计入预算**: 这是关键差异
- **finish_reason="length"**: 可能表示推理消耗全部预算，输出为空
- **不确定性**: 无法提前知道推理会消耗多少 tokens

### 对比表

| 特性 | OpenAI | Anthropic | 智谱 GLM-5 |
|------|--------|-----------|-----------|
| max_tokens 限制对象 | 仅输出内容 | 仅输出内容 | 推理 + 输出总和 |
| 推理 tokens 是否计入 | ❌（o1 除外） | ❌ | ✅ |
| 行为可预测性 | 高 | 高 | 低 |
| 空输出风险 | 低 | 低 | 高 |
| API 响应中的 reasoning_tokens 字段 | 有（o1） | 无 | 有 |

---

## 5. GLM-5 生成长文件的最佳实践配置

### 方案 1: 禁用思考模式（推荐用于非复杂任务）

根据 MemOS PR #1310 的解决方案，在请求中注入：

```json
{
  "thinking": {
    "type": "disabled"
  }
}
```

**优点**：
- 完全避免推理 tokens 消耗预算
- 行为与其他模型 API 一致
- 确保所有 max_tokens 用于实际输出

**缺点**：
- 失去 GLM-5 的推理能力
- 对于复杂任务，输出质量可能下降

**适用场景**：
- 简单问答
- 文本生成（如生成长文件）
- 不需要深度推理的任务

### 方案 2: 增加 max_tokens 并设置合理的 context_length

```yaml
# config.yaml
model:
  context_length: 32000  # 或更低，根据实际需求
  max_tokens: 8000       # 为推理预留空间
compression:
  threshold: 0.5         # 更早触发压缩
  target_ratio: 0.6      # 压缩到 60%
```

**优点**：
- 保留推理能力
- 通过增加预算降低空输出风险

**缺点**：
- 仍然存在不确定性
- 不保证推理不会消耗全部预算
- 增加成本

**适用场景**：
- 需要推理能力的复杂任务
- 能够承受偶尔的重试

### 方案 3: 检测并重试（需要代码实现）

实现以下逻辑：

```python
def call_glm5_with_retry(messages, max_tokens=16000, max_retries=3):
    for attempt in range(max_retries):
        response = client.chat.completions.create(
            model="glm-5-turbo",
            messages=messages,
            max_tokens=max_tokens
        )
        
        # 检查是否为仅推理响应
        if (response.choices[0].finish_reason == "length" and
            response.choices[0].message.content == "" and
            response.choices[0].message.reasoning_content):
            # 增加预算并重试
            max_tokens = int(max_tokens * 1.5)
            continue
        
        return response
    
    # 所有重试都失败
    raise Exception("GLM-5 reasoning exhausted all token budgets")
```

**优点**：
- 保留推理能力
- 自动适应推理 token 消耗

**缺点**：
- 需要自定义实现
- 增加延迟和成本
- 可能需要多次重试

**适用场景**：
- 对输出质量要求高
- 能够承受额外延迟和成本
- 有技术能力实现自定义逻辑

### 方案 4: 使用非思考模型（如 GLM-4-Flash）

如果不需要推理能力，可以使用：

- **GLM-4-Flash**: 更快、更便宜、无推理 tokens 问题
- **glm-4-plus**: 更大上下文、稳定输出

**适用场景**：
- 不需要推理能力
- 需要稳定可预测的输出
- 成本敏感

### 推荐配置（生成长文件）

对于生成长文件的任务，推荐以下配置：

```yaml
# config.yaml
provider:
  type: custom
  base_url: https://open.bigmodel.cn/api/paas/v4
  api_key: ${ZHIPU_API_KEY}
  
model:
  name: glm-4-flash  # 或禁用思考的 glm-5-turbo
  context_length: 32000
  max_tokens: 8000
  
  # 禁用思考模式（如果使用 glm-5-turbo）
  extra_body:
    thinking:
      type: disabled

compression:
  threshold: 0.4  # 40% 时触发
  target_ratio: 0.6  # 压缩到 60%
  
  # 或者使用主动压缩
  proactive: true
  target_tokens: 20000  # 目标上下文大小
```

---

## 6. 技术要点总结

### 关键发现

1. **reasoning_tokens 计入 max_tokens**: 这是 GLM-5 与其他模型 API 的最大差异
2. **默认 max_tokens 约 16K**: 根据实际测试，不是文档中可能提到的更高值
3. **有效上下文约 32K**: 考虑到输入 + 推理 + 输出的总和
4. **finish_reason="length" + content=""**: 典型的推理耗尽信号

### 风险提示

- ⚠️ **风险高**: 在 Agent 框架中使用 GLM-5-Turbo 默认配置，极大概率遇到空输出问题
- ⚠️ **渐进性**: 问题随对话长度增加而恶化
- ⚠️ **恢复困难**: 标准重试机制可能加剧问题

### 最佳实践

1. **禁用思考模式**（用于简单任务）
2. **设置保守的 context_length**（32K 或更低）
3. **使用更低的 compression threshold**（0.4 或 0.5）
4. **实现推理检测和自动重试**（如果保留思考模式）
5. **考虑使用 GLM-4-Flash**（如果不需要推理）

---

## 7. 参考来源

### GitHub Issues & PRs

1. **[Hermes Agent #9344](https://github.com/NousResearch/hermes-agent/issues/9344)**: Thinking model (glm-5-turbo) reasoning tokens exhaust output budget, producing empty responses with no recovery path
   - 发布时间: 2026-04-14
   - 状态: Open
   - 标签: P1, type/bug, comp/agent, provider/zai

2. **[MemOS #1310](https://github.com/MemTensor/MemOS/pull/1310)**: fix: disable zhipu thinking mode to prevent summarizer output truncation
   - 合并时间: 2026-03-23
   - 状态: Merged

3. **[glm-acp-agent #1](https://github.com/stefandevo/glm-acp-agent/pull/1)**: Implement GLM ACP Agent — native TypeScript ACP protocol with Zhipu AI GLM backend
   - 提到 GLM-5.1 thinking mode

### 相关代码仓库

- **stefandevo/glm-acp-agent**: GLM Native ACP Agent 实现
- **MemTensor/MemOS**: 智谱思考模式的修复方案
- **NousResearch/hermes-agent**: Agent 框架中的 GLM-5 问题

---

## 8. 建议后续行动

1. **立即行动**: 为 GLM-5-Turbo 配置禁用思考模式或使用更保守的参数
2. **短期**: 实现推理 token 检测和自动重试机制
3. **长期**: 与智谱 API 团队沟通，了解是否有计划改进 API 行为或提供更好的控制选项

---

*报告生成时间: 2026-06-09*  
*数据来源: GitHub Issues, PRs, 和相关项目代码*
