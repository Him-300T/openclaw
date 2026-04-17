# 文章笔记：在OpenClaw实测谷歌开源大模型Gemma 4

## 基本信息
- **文章标题**：🦞在OpenClaw实测谷歌开源大模型Gemma 4！256K上下文+多模态输入，小龙虾里实战测试多Agent协作与浏览器自动化全流程！31B参数碾压其他开源大模型！五大维度深度评分结果出人意料
- **文章链接**：https://www.aivi.fyi/llms/Gemm4-in-OpenClaw
- **阅读时间**：1分钟
- **阅读日期**：2026-04-17
- **笔记作者**：R-Overlord 👑

## 核心内容

### 目标
将OpenClaw配置为使用`gemma-4-31b-it`作为默认模型，并确保其正常工作。

## 遇到的问题

### 三层兼容性问题

#### 1. OpenRouter / HuggingFace Router不支持Gemma 4的tool calling
- OpenClaw的agent依赖工具调用
- 这些接入方式对Gemma 4不可用

#### 2. OpenClaw内置的google/* provider不包含Gemma模型
- 直接使用内置Google provider时
- `gemma-4-31b-it`会被识别为未知模型

#### 3. Gemma 4流式响应包含thinking token
- 响应中包含`"thought": true`的thinking/reasoning token
- OpenClaw响应解析器会尝试按结构化reasoning输出处理
- 触发`MALFORMED_RESPONSE`错误

## 解决方案

### 核心方案：使用自定义google-ai provider
1. 使用Google Generative AI原生API格式
2. 在模型定义中加入`"reasoning": false`
3. 跳过对reasoning/thinking token的解析

## 配置步骤

### 步骤1：添加自定义provider
在`~/.openclaw/openclaw.json`的`models.providers`下加入：

```json
"google-ai": {
  "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
  "api": "google-generative-ai",
  "authHeader": true,
  "models": [
    {
      "id": "gemma-4-31b-it",
      "name": "Gemma 4 31B IT",
      "reasoning": false,
      "input": ["text", "image"],
      "cost": {
        "input": 0,
        "output": 0
      },
      "contextWindow": 262144,
      "maxTokens": 32768
    }
  ]
}
```

### 步骤2：设置默认模型
在`~/.openclaw/openclaw.json`中配置：

```json
// agents.defaults.model
"primary": "google-ai/gemma-4-31b-it",
"fallbacks": ["minimax/MiniMax-M2.7", "minimax-portal/MiniMax-M2.1"]

// agents.list[0]（主agent）
"primary": "google-ai/gemma-4-31b-it"

// agents.defaults.models（模型目录项）
"google-ai/gemma-4-31b-it": {
  "alias": "gemma4"
}
```

### 步骤3：添加认证配置
在`~/.openclaw/agents/main/agent/auth-profiles.json`中添加：

```json
"google-ai:default": {
  "type": "api_key",
  "provider": "google-ai",
  "key": "<GOOGLE_AI_STUDIO_API_KEY>"
}
```

### 步骤4：配置API Key
在`~/.openclaw/.env`中设置：

```
GOOGLE_AI_API_KEY=AIza...
```

## 技术要点

### Gemma 4模型特性
- **参数**：31B
- **上下文窗口**：256K tokens（262144）
- **最大输出**：32K tokens（32768）
- **多模态输入**：支持文本和图像
- **推理模式**：需要设置为`false`避免解析错误

### 成本信息
- 文章中显示成本为0（可能测试期间免费或作者配置）
- 实际使用时需要确认Google AI Studio的定价

## 关键配置细节

### 1. reasoning: false的重要性
- Gemma 4的流式响应包含thinking token
- OpenClaw会尝试解析为结构化reasoning输出
- 设置为false避免`MALFORMED_RESPONSE`错误

### 2. 自定义provider的必要性
- 内置google provider不包含Gemma模型
- 需要自定义provider直连Google AI Studio
- 避免通过OpenRouter/HuggingFace Router中转

### 3. API端点配置
- **baseUrl**: `https://generativelanguage.googleapis.com/v1beta`
- **api**: `google-generative-ai`
- **authHeader**: true（使用认证头）

## 实践建议

### 适用场景
- 需要超大上下文（256K）的任务
- 多模态输入（文本+图像）处理
- 开源大模型爱好者/研究者
- 需要免费或低成本模型的场景

### 配置注意事项
1. **获取API Key**：需要Google AI Studio账号和API Key
2. **成本确认**：确认实际使用成本，文章中显示为0可能不准确
3. **性能测试**：31B参数模型可能需要较高计算资源
4. **兼容性验证**：测试工具调用功能是否正常

## 与其他模型对比

### 相比当前使用的DeepSeek
- **上下文**：256K vs 128K（Gemma 4更大）
- **多模态**：支持图像输入（Gemma 4优势）
- **开源**：Gemma 4是开源模型
- **成本**：需要具体比较（DeepSeek当前$0.28/1M输入）

### 相比MiniMax
- **参数**：31B vs 可能不同
- **上下文**：256K vs 通常较小
- **开源**：Gemma 4开源，MiniMax可能闭源

## 潜在问题与解决

### 可能遇到的问题
1. **API Key限制**：Google AI Studio可能有使用限制
2. **响应速度**：31B模型可能响应较慢
3. **工具调用**：需要验证工具调用功能
4. **中文支持**：需要测试中文处理能力

### 解决方案
1. 使用fallback机制配置备用模型
2. 监控API使用情况和成本
3. 测试关键功能确保可用性
4. 根据实际需求选择模型

## 学习收获

### 技术知识
1. 掌握了OpenClaw添加自定义provider的方法
2. 理解了Gemma 4模型的特性和配置要求
3. 学会了解决模型兼容性问题的方法

### 配置技能
1. 多层级配置：provider、模型、认证、环境变量
2. 故障排除：识别和解决三层兼容性问题
3. 优化配置：设置reasoning避免解析错误

### 认知提升
1. 认识到开源大模型在OpenClaw中的集成方式
2. 理解了不同模型provider的差异和配置要求
3. 掌握了模型切换和配置的最佳实践

## 后续行动

### 立即行动
1. 评估是否尝试Gemma 4模型
2. 如果需要，按照配置步骤操作
3. 测试模型性能和功能

### 长期考虑
1. 关注Gemma模型更新
2. 比较不同开源模型的性价比
3. 根据任务需求灵活选择模型

## 关联资源
- **B站视频**：https://www.bilibili.com/video/BV1Ff9gBwESC/
- **YouTube视频**：https://youtu.be/GJYQ9Aw7wL8
- **Google AI Studio**：需要获取API Key

## 技术总结

### 解决的问题
1. OpenRouter/HuggingFace Router不支持Gemma 4工具调用
2. 内置provider不包含Gemma模型
3. Gemma 4的thinking token导致解析错误

### 提供的方案
1. 自定义google-ai provider直连API
2. 设置reasoning: false避免解析错误
3. 完整的多层级配置方案

### 关键配置
1. 自定义provider定义
2. 模型参数配置
3. 认证和环境变量设置

---

**笔记状态**：✅ 已完成深度阅读和总结  
**实践价值**：⭐⭐⭐（对需要开源大模型或超大上下文的用户）  
**技术难度**：⭐⭐⭐⭐（配置较复杂，涉及多层级）  
**当前相关性**：⭐⭐（我们当前使用DeepSeek，但了解配置方法有价值）