# GLM-5-Turbo 研究总结

## 核心发现

### 1. Max Tokens 限制

- **默认 max_tokens**: ~16,000 tokens
- **有效上下文窗口**: 32,000 tokens（实际可用）
- **理论上下文**: 128,000 tokens（但实际无法完全利用）

### 2. 关键问题：reasoning_tokens 计入 max_tokens

这是 GLM-5 与其他模型 API 最大的区别：

```
max_tokens = reasoning_tokens + output_tokens
```

当推理消耗全部预算时：
- `content = ""`
- `finish_reason = "length"`
- 用户体验：收到空响应

### 3. 智谱 API 截断信号格式

```json
{
  "choices": [{
    "message": {
      "content": "",
      "reasoning_content": "思考内容..."
    },
    "finish_reason": "length"
  }],
  "usage": {
    "completion_tokens": 16000,
    "reasoning_tokens": 16000
  }
}
```

### 4. Hermes Agent 中的问题（Issue #9344）

**现象**：
- 对话中途返回 `(empty)` 响应
- 随对话长度增加而恶化
- 确定性、渐进性故障

**根本原因**：
1. 推理 tokens 消耗全部输出预算
2. Prefill 恢复机制适得其反（增加上下文 → 更多推理 → 仍为空）
3. 压缩触发太晚（在响应之后）

**Workaround**：
```yaml
model:
  context_length: 32000
compression:
  threshold: 0.4
```

### 5. 与其他 API 的差异

| 特性 | OpenAI | Anthropic | 智谱 GLM-5 |
|------|--------|-----------|-----------|
| max_tokens 限制 | 仅输出 | 仅输出 | 推理+输出总和 |
| 推理计入 | ❌ | ❌ | ✅ |
| 可预测性 | 高 | 高 | 低 |
| 空输出风险 | 低 | 低 | 高 |

### 6. 最佳实践（4 个方案）

#### 方案 1：禁用思考模式 ⭐ 推荐
```json
{"thinking": {"type": "disabled"}}
```
- ✅ 完全避免问题
- ❌ 失去推理能力
- 适用：文本生成、简单任务

#### 方案 2：保守配置
```yaml
model:
  context_length: 32000
  max_tokens: 8000
compression:
  threshold: 0.4
```
- ✅ 保留推理能力
- ⚠️ 仍存在不确定性
- 适用：需要推理的任务

#### 方案 3：检测并重试
```python
if content == "" and reasoning_content and finish_reason == "length":
    max_tokens *= 1.5
    retry
```
- ✅ 自动适应
- ❌ 需要自定义实现
- 适用：有技术能力的团队

#### 方案 4：使用非思考模型
- GLM-4-Flash：快、便宜、稳定
- 适用：不需要推理的场景

### 7. 推荐配置（生成长文件）

```yaml
provider:
  base_url: https://open.bigmodel.cn/api/paas/v4
  api_key: ${ZHIPU_API_KEY}
  
model:
  name: glm-4-flash  # 或 glm-5-turbo + disabled thinking
  context_length: 32000
  max_tokens: 8000
  extra_body:
    thinking:
      type: disabled  # 如果用 glm-5

compression:
  threshold: 0.4
  target_ratio: 0.6
```

## 风险评估

⚠️ **高风险**：在 Agent 框架中使用 GLM-5-Turbo 默认配置，极大概率遇到空输出问题

⚠️ **渐进性**：问题随对话长度增加而恶化

⚠️ **恢复困难**：标准重试机制可能加剧问题

## 立即行动建议

1. 为 GLM-5-Turbo 配置禁用思考模式
2. 或使用更保守的 context_length (32K)
3. 实现推理检测和自动重试
4. 考虑使用 GLM-4-Flash（如果不需要推理）

## 参考来源

- Hermes Agent Issue #9344
- MemOS PR #1310
- glm-acp-agent PR #1
