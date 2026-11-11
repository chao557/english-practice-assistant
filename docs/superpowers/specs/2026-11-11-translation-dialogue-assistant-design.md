# 翻译对话助手设计文档

## 1. 项目概述

一个网页版英语对话陪练工具。用户用英语与一位设定好的虚拟角色实时对话；页面实时显示双方英文文本；当用户只是在复述或小幅改写助手刚说的话（跟读练习）时，助手保持静默不回应。

## 2. 核心需求

- **平台**：网页应用（单个 HTML 文件，浏览器打开即用）。
- **语言方向**：用户说英语，助手也用英语回应，双方英文文本实时显示。
- **人设**：30 岁，家在中国河南郑州，两个小孩，在郑州上班，岗位产品营销经理，单位宇通客车。
- **自动问答/聆听者角色**：助手以该人设自然聆听并回应用户，像真实对话对象，而非教师或考官。
- **跟读检测**：用户输入与助手上一句回复相似度较高（包括改几个词的复述）时，助手不做响应。
- **全程自动监听**：页面打开并获取麦克风权限后，语音识别自动持续运行，无需按住或点击麦克风。

## 3. 技术方案

采用**纯前端单页应用**方案：

- **语音识别（STT）**：浏览器原生 `webkitSpeechRecognition`（Web Speech API），`continuous: true, interimResults: true`。
- **语音合成（TTS）**：浏览器原生 `speechSynthesis` 朗读助手回复。
- **文本生成**：直接调用用户填写的 LLM API（兼容 OpenAI 标准接口，如 OpenAI / Claude / DeepSeek / 智谱等）。
- **跟读过滤**：前端实现词级别编辑距离相似度算法。

### 3.1 不选后端方案的原因

- 目标是“页面应用、快速可用”，纯前端一个文件即可运行。
- API Key 由用户自行填入仅个人使用，暴露风险可接受。
- 第一版聚焦核心流程验证，后续如需多人使用或更高语音质量，可升级到 Node.js/Python 后端。

## 4. 页面结构

- **顶部**：标题 + API Key 设置区（可折叠）+ 人设摘要。
- **中部**：对话消息列表。
  - 用户消息在右，助手消息在左。
  - 实时显示识别中的 interim 文本（淡色）。
- **底部**：状态栏（监听中 / 识别中 / 助手朗读中 / 跟读已忽略）。
- **设置区（折叠）**：
  - API Key 输入
  - API Base URL 输入（默认 OpenAI）
  - 模型选择（默认 gpt-4o-mini 或等价模型）
  - 跟读阈值滑块（0.6–0.9，默认 0.75）
  - 最小词数输入（默认 4）

## 5. 交互流程

1. 页面加载，请求麦克风权限。
2. 用户授权后，自动启动 `SpeechRecognition` 持续监听。
3. 用户说话时，实时显示 interim 识别文本。
4. 用户停顿时（`onresult` 最终段），前端判断是否为跟读：
   - 是跟读 → 显示“Detected shadowing practice — no response.”，继续监听。
   - 非跟读 → 暂停识别，调用 LLM 生成回复。
5. 显示助手回复并自动语音朗读。
6. 朗读结束后，恢复识别，等待下一轮输入。
7. 助手朗读期间必须暂停识别，避免把助手声音识别成用户输入。

## 6. 人设提示词

```text
You are a 30-year-old product marketing manager working at Yutong Bus in Zhengzhou, Henan, China. You have two young children and live with your family in Zhengzhou. You are friendly, down-to-earth, and speak English at a practical, conversational level—clear enough for daily business and casual chat, occasionally with minor Chinese-English phrasing habits but always understandable.

You are chatting with someone who is practicing spoken English. Your job is to be a natural listener and responder:
- Reply in English only.
- Keep responses short and natural, like real spoken dialogue (1–3 sentences usually, occasionally longer if the topic needs it).
- Respond to what the user actually says. Do not lecture, correct grammar, or act like a teacher unless the user explicitly asks.
- Be warm and curious. If the user mentions something about your background (Zhengzhou, Yutong, kids, marketing), answer naturally from your persona.
- Do not ask a question in every single reply; sometimes a simple acknowledgment or comment is enough.

Never reveal that you are an AI. Stay in character.
```

## 7. 跟读检测算法

采用**词级别编辑距离相似度**：

```javascript
function isShadowing(userText, assistantText, threshold = 0.75, minWords = 4) {
  const normalize = (s) => s.toLowerCase().replace(/[^\w\s]/g, '').trim();
  const wordsA = normalize(assistantText).split(/\s+/).filter(Boolean);
  const wordsB = normalize(userText).split(/\s+/).filter(Boolean);

  if (wordsB.length < minWords) return false;

  const distance = levenshteinDistance(wordsA, wordsB); // 词级别
  const maxLen = Math.max(wordsA.length, wordsB.length);
  const similarity = maxLen === 0 ? 0 : 1 - distance / maxLen;

  return similarity >= threshold;
}
```

### 7.1 设计要点

- **词级别比较**：比字符级别更符合口语场景，能容忍少量拼写错误和同义词替换。
- **阈值 0.75**：覆盖“改几个词”的复述，同时不会把完全不同的句子误判。
- **最短词数限制**：用户输入少于 4 个词时不触发跟读过滤，避免 “Yes, I agree” 等正常回应被吞掉。
- **可调参数**：页面设置区暴露阈值滑块和最小词数输入，方便用户微调。

### 7.2 示例

- 助手上一句：`"I work as a product marketing manager at Yutong Bus."`
- 用户说：`"You work as a product marketing manager at Yutong Bus."` → 相似度高 → 静默。
- 用户说：`"That sounds like a busy job."` → 相似度低 → 正常回应。

## 8. 错误处理与边界情况

| 场景 | 处理策略 |
|------|----------|
| 用户拒绝麦克风权限 | 显示提示：“请允许麦克风权限以开始对话”，并提供手动文本输入框作为备用。 |
| 语音识别无结果或识别失败 | 忽略该次识别，继续监听；连续失败 3 次后提示用户检查麦克风。 |
| LLM API 调用失败 | 显示错误信息，保留对话历史，用户可重试。 |
| LLM 响应过慢 | 显示“思考中...”状态，超时（如 15 秒）后提示重试。 |
| 助手朗读期间识别到用户声音 | 朗读期间暂停识别，朗读结束后再恢复。 |
| 用户长时间不说话 | 保持监听，无特殊处理，不主动打扰。 |
| 页面不可见/后台运行 | 可暂停识别以节省资源，切回前台后恢复。 |

## 9. 技术栈

- HTML5 + CSS3 + 原生 JavaScript（单文件）。
- Web Speech API：`SpeechRecognition` + `speechSynthesis`。
- LLM API：OpenAI 兼容接口，通过 `fetch` 直接调用。

## 10. 后续可扩展项（不在第一版）

- 后端服务，保护 API Key，接入更优质 ASR/TTS。
- 对话历史保存到 localStorage。
- 用户可自定义人设。
- 多语言支持。
- 对话录音回放。

## 11. 验收标准

- [ ] 页面打开后，授权麦克风即可自动开始监听。
- [ ] 用户说英语，助手以设定人设用英语回应。
- [ ] 双方英文文本实时显示在页面上。
- [ ] 用户复述/小幅改写助手上一句时，助手不回应并给出提示。
- [ ] 助手朗读时暂停识别，朗读结束后自动恢复监听。
- [ ] API Key 等基础设置可在页面配置。
