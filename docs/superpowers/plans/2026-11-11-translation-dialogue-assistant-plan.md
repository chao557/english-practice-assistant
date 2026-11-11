# 翻译对话助手实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个单文件网页版英语对话陪练助手，用户授权麦克风后自动监听，助手以固定人设回应，并能识别跟读重复保持静默。

**Architecture:** 纯前端单页应用，所有 HTML/CSS/JS 集中在 `index.html`。使用浏览器 Web Speech API 做语音听写与朗读，通过 `fetch` 调用用户配置的 OpenAI 兼容 LLM API，前端实现词级编辑距离跟读检测。

**Tech Stack:** HTML5, CSS3, 原生 JavaScript, Web Speech API, OpenAI 兼容 LLM API。

**Spec:** [docs/superpowers/specs/2026-11-11-translation-dialogue-assistant-design.md](../specs/2026-11-11-translation-dialogue-assistant-design.md)

## Global Constraints

- 单文件应用：`index.html` 包含全部代码。
- 仅使用浏览器原生 API 与标准 `fetch`。
- 用户需自行填入 LLM API Key；第一版不建后端。
- 英文界面为主，但提示文案可中英并存以方便测试。
- 必须在 Chrome/Edge 测试（Web Speech API 支持最好）。

---

## Task 1: 创建项目骨架与基础 UI

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: 页面 DOM 结构，供 Task 2-5 挂载事件与动态内容。

- [ ] **Step 1: 创建 `index.html` 基础结构**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>English Practice Assistant</title>
  <style>
    /* 基础样式占位，将在 Task 5 完善 */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, -apple-system, sans-serif; background: #f5f5f5; color: #333; height: 100vh; display: flex; flex-direction: column; }
    header { background: #2c3e50; color: white; padding: 12px 16px; }
    main { flex: 1; overflow-y: auto; padding: 16px; }
    .message { max-width: 75%; padding: 10px 14px; margin: 8px 0; border-radius: 12px; line-height: 1.5; }
    .user { background: #dbeafe; align-self: flex-end; margin-left: auto; }
    .assistant { background: white; border: 1px solid #e5e7eb; }
    .interim { opacity: 0.6; }
    .shadowing-hint { color: #6b7280; font-size: 12px; text-align: center; margin: 4px 0; }
    footer { padding: 12px 16px; background: white; border-top: 1px solid #e5e7eb; display: flex; gap: 8px; align-items: center; }
    #status { font-size: 12px; color: #6b7280; min-width: 120px; }
  </style>
</head>
<body>
  <header>
    <h1>English Practice Assistant</h1>
    <p>Listener: 30-year-old product marketing manager at Yutong Bus, Zhengzhou, two kids.</p>
  </header>
  <main id="chat"></main>
  <footer>
    <span id="status">Initializing...</span>
    <button id="settingsBtn">Settings</button>
  </footer>

  <!-- 设置面板 -->
  <dialog id="settingsDialog">
    <form method="dialog">
      <h2>Settings</h2>
      <label>API Key: <input id="apiKey" type="password" placeholder="sk-..." style="width:100%" /></label>
      <label>Base URL: <input id="baseUrl" type="url" value="https://api.openai.com/v1" style="width:100%" /></label>
      <label>Model: <input id="model" value="gpt-4o-mini" style="width:100%" /></label>
      <label>Shadowing Threshold: <input id="threshold" type="range" min="0.6" max="0.9" step="0.05" value="0.75" /></label>
      <label>Min Words: <input id="minWords" type="number" value="4" min="1" /></label>
      <div>
        <button value="save">Save</button>
        <button value="cancel">Cancel</button>
      </div>
    </form>
  </dialog>

  <script>
    // JS 占位，将在后续 Task 填充
    console.log('App loaded');
  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中打开 `index.html` 验证页面能正常显示**

手动测试：页面标题、人设说明、设置按钮可见。

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: scaffold single-page English practice assistant"
```

---

## Task 2: 实现语音识别（STT）自动监听

**Files:**
- Modify: `index.html` 的 `<script>` 区域

**Interfaces:**
- Produces: 全局 `recognition` 实例，`startListening()` 与 `stopListening()` 函数。
- Produces: `appendMessage(role, text, isInterim = false)` 用于后续任务。

- [ ] **Step 1: 在 `<script>` 中检测并初始化 SpeechRecognition**

```javascript
const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
if (!SpeechRecognition) {
  updateStatus('Speech API not supported. Use Chrome/Edge.');
  throw new Error('SpeechRecognition not supported');
}

const recognition = new SpeechRecognition();
recognition.continuous = true;
recognition.interimResults = true;
recognition.lang = 'en-US';

let isListening = false;
let isAssistantSpeaking = false;

function startListening() {
  if (isListening || isAssistantSpeaking) return;
  try {
    recognition.start();
    isListening = true;
    updateStatus('Listening...');
  } catch (e) {
    console.error('Start failed', e);
  }
}

function stopListening() {
  if (!isListening) return;
  recognition.stop();
  isListening = false;
  updateStatus('Paused');
}
```

- [ ] **Step 2: 实现实时文本显示与最终结果处理**

```javascript
let currentInterimId = null;

recognition.onresult = (event) => {
  const results = event.results;
  const last = results[results.length - 1];
  const transcript = last[0].transcript.trim();

  if (last.isFinal) {
    removeInterim();
    handleUserInput(transcript);
  } else {
    showInterim(transcript);
  }
};

recognition.onerror = (event) => {
  console.error('Speech error', event.error);
  updateStatus(`Error: ${event.error}`);
  isListening = false;
  setTimeout(startListening, 1000);
};

recognition.onend = () => {
  isListening = false;
  if (!isAssistantSpeaking) startListening();
};

function showInterim(text) {
  removeInterim();
  currentInterimId = appendMessage('user', text, true);
}

function removeInterim() {
  if (currentInterimId) {
    const el = document.getElementById(currentInterimId);
    if (el) el.remove();
    currentInterimId = null;
  }
}
```

- [ ] **Step 3: 页面加载后请求权限并开始监听**

```javascript
async function initMicrophone() {
  try {
    await navigator.mediaDevices.getUserMedia({ audio: true });
    startListening();
  } catch (err) {
    updateStatus('Microphone permission denied.');
  }
}

initMicrophone();
```

- [ ] **Step 4: 实现 `appendMessage` 与 `updateStatus` 辅助函数**

```javascript
function appendMessage(role, text, isInterim = false) {
  const chat = document.getElementById('chat');
  const id = `msg-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`;
  const div = document.createElement('div');
  div.id = id;
  div.className = `message ${role}${isInterim ? ' interim' : ''}`;
  div.textContent = text;
  chat.appendChild(div);
  chat.scrollTop = chat.scrollHeight;
  return id;
}

function updateStatus(text) {
  document.getElementById('status').textContent = text;
}

// 占位：后续 Task 实现
function handleUserInput(text) {
  console.log('User said:', text);
  appendMessage('user', text);
}
```

- [ ] **Step 5: 在 Chrome/Edge 中打开页面，允许麦克风，说几句英文验证文本实时显示**

手动测试：interim 文本出现，停止说话后变为最终消息。

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: auto speech recognition and real-time transcript display"
```

---

## Task 3: 实现跟读检测算法

**Files:**
- Modify: `index.html` 的 `<script>` 区域

**Interfaces:**
- Produces: `isShadowing(userText, assistantText, threshold, minWords)` 函数。
- Consumes: 助手上一句回复文本（后续 Task 维护）。

- [ ] **Step 1: 实现词级 Levenshtein 距离函数**

```javascript
function levenshteinDistance(a, b) {
  const matrix = Array.from({ length: a.length + 1 }, (_, i) => [i]);
  for (let j = 1; j <= b.length; j++) matrix[0][j] = j;

  for (let i = 1; i <= a.length; i++) {
    for (let j = 1; j <= b.length; j++) {
      const cost = a[i - 1] === b[j - 1] ? 0 : 1;
      matrix[i][j] = Math.min(
        matrix[i - 1][j] + 1,
        matrix[i][j - 1] + 1,
        matrix[i - 1][j - 1] + cost
      );
    }
  }
  return matrix[a.length][b.length];
}

function normalizeText(text) {
  return text.toLowerCase().replace(/[^\w\s]/g, '').trim();
}

function isShadowing(userText, assistantText, threshold = 0.75, minWords = 4) {
  if (!assistantText || !userText) return false;
  const wordsA = normalizeText(assistantText).split(/\s+/).filter(Boolean);
  const wordsB = normalizeText(userText).split(/\s+/).filter(Boolean);

  if (wordsB.length < minWords) return false;

  const distance = levenshteinDistance(wordsA, wordsB);
  const maxLen = Math.max(wordsA.length, wordsB.length);
  const similarity = maxLen === 0 ? 0 : 1 - distance / maxLen;
  return similarity >= threshold;
}
```

- [ ] **Step 2: 在浏览器控制台测试几个用例**

```javascript
const prev = "I work as a product marketing manager at Yutong Bus.";
console.log(isShadowing("You work as a product marketing manager at Yutong Bus.", prev)); // true
console.log(isShadowing("That sounds like a busy job.", prev)); // false
console.log(isShadowing("Ok", prev)); // false
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: word-level shadowing detection"
```

---

## Task 4: 实现 LLM 调用与人设提示词

**Files:**
- Modify: `index.html` 的 `<script>` 区域

**Interfaces:**
- Produces: `sendToLLM(userText)` 函数，返回助手的英文回复字符串。
- Produces: `lastAssistantText` 变量，保存助手最后一句完整回复。
- Consumes: 设置面板中的 `apiKey`、`baseUrl`、`model`。

- [ ] **Step 1: 定义人设系统提示词**

```javascript
const SYSTEM_PROMPT = `You are a 30-year-old product marketing manager working at Yutong Bus in Zhengzhou, Henan, China. You have two young children and live with your family in Zhengzhou. You are friendly, down-to-earth, and speak English at a practical, conversational level—clear enough for daily business and casual chat, occasionally with minor Chinese-English phrasing habits but always understandable.

You are chatting with someone who is practicing spoken English. Your job is to be a natural listener and responder:
- Reply in English only.
- Keep responses short and natural, like real spoken dialogue (1–3 sentences usually, occasionally longer if the topic needs it).
- Respond to what the user actually says. Do not lecture, correct grammar, or act like a teacher unless the user explicitly asks.
- Be warm and curious. If the user mentions something about your background (Zhengzhou, Yutong, kids, marketing), answer naturally from your persona.
- Do not ask a question in every single reply; sometimes a simple acknowledgment or comment is enough.

Never reveal that you are an AI. Stay in character.`;
```

- [ ] **Step 2: 实现 LLM 调用函数**

```javascript
async function sendToLLM(userText) {
  const apiKey = document.getElementById('apiKey').value.trim();
  const baseUrl = document.getElementById('baseUrl').value.trim() || 'https://api.openai.com/v1';
  const model = document.getElementById('model').value.trim() || 'gpt-4o-mini';

  if (!apiKey) {
    updateStatus('Please set API Key in Settings.');
    return null;
  }

  updateStatus('Thinking...');

  const response = await fetch(`${baseUrl}/chat/completions`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        { role: 'user', content: userText }
      ],
      temperature: 0.8,
      max_tokens: 150
    })
  });

  if (!response.ok) {
    const err = await response.text();
    throw new Error(`LLM error ${response.status}: ${err}`);
  }

  const data = await response.json();
  return data.choices?.[0]?.message?.content?.trim() || '';
}
```

- [ ] **Step 3: 实现 `handleUserInput` 主流程**

```javascript
let lastAssistantText = '';

async function handleUserInput(text) {
  if (!text) return;

  const threshold = parseFloat(document.getElementById('threshold').value) || 0.75;
  const minWords = parseInt(document.getElementById('minWords').value, 10) || 4;

  if (lastAssistantText && isShadowing(text, lastAssistantText, threshold, minWords)) {
    showShadowingHint();
    return;
  }

  stopListening();

  try {
    const reply = await sendToLLM(text);
    if (!reply) return;

    lastAssistantText = reply;
    appendMessage('assistant', reply);
    speak(reply);
  } catch (err) {
    console.error(err);
    updateStatus(`Error: ${err.message}`);
    startListening();
  }
}

function showShadowingHint() {
  const chat = document.getElementById('chat');
  const div = document.createElement('div');
  div.className = 'shadowing-hint';
  div.textContent = 'Detected shadowing practice — no response.';
  chat.appendChild(div);
  chat.scrollTop = chat.scrollHeight;
  updateStatus('Shadowing detected');
  setTimeout(() => updateStatus('Listening...'), 1500);
}
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: LLM integration with persona prompt"
```

---

## Task 5: 实现语音合成（TTS）与监听恢复

**Files:**
- Modify: `index.html` 的 `<script>` 区域

**Interfaces:**
- Produces: `speak(text)` 函数，朗读完成后自动恢复监听。
- Consumes: `stopListening()`、`startListening()`。

- [ ] **Step 1: 实现 `speak` 函数**

```javascript
function speak(text) {
  if (!window.speechSynthesis) {
    updateStatus('TTS not supported.');
    startListening();
    return;
  }

  isAssistantSpeaking = true;
  stopListening();
  updateStatus('Speaking...');

  const utter = new SpeechSynthesisUtterance(text);
  utter.lang = 'en-US';
  utter.rate = 1;
  utter.pitch = 1;

  utter.onend = () => {
    isAssistantSpeaking = false;
    updateStatus('Listening...');
    startListening();
  };

  utter.onerror = (err) => {
    console.error('TTS error', err);
    isAssistantSpeaking = false;
    startListening();
  };

  window.speechSynthesis.speak(utter);
}
```

- [ ] **Step 2: 确保朗读期间 `recognition.onend` 不会自动重启监听**

已在 Task 2 的 `recognition.onend` 中检查 `isAssistantSpeaking`。

- [ ] **Step 3: 在浏览器中测试完整流程**

手动测试：
1. 填入有效 API Key。
2. 说 "Hello, where are you from?"
3. 助手应以人设回应。
4. 复述助手的话，应触发跟读提示且无新回复。
5. 说新话题，助手继续回应。

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: text-to-speech and auto-resume listening"
```

---

## Task 6: 完善 UI、设置持久化与错误提示

**Files:**
- Modify: `index.html`

**Interfaces:**
- Produces: 美观可折叠设置面板、localStorage 持久化、更明显的状态提示。

- [ ] **Step 1: 实现设置面板打开/保存逻辑**

```javascript
const settingsBtn = document.getElementById('settingsBtn');
const settingsDialog = document.getElementById('settingsDialog');

settingsBtn.addEventListener('click', () => {
  settingsDialog.showModal();
});

settingsDialog.addEventListener('close', () => {
  if (settingsDialog.returnValue === 'save') {
    saveSettings();
  }
});

function saveSettings() {
  const settings = {
    apiKey: document.getElementById('apiKey').value,
    baseUrl: document.getElementById('baseUrl').value,
    model: document.getElementById('model').value,
    threshold: document.getElementById('threshold').value,
    minWords: document.getElementById('minWords').value
  };
  localStorage.setItem('englishPracticeSettings', JSON.stringify(settings));
  updateStatus('Settings saved');
  setTimeout(startListening, 500);
}

function loadSettings() {
  const raw = localStorage.getItem('englishPracticeSettings');
  if (!raw) return;
  try {
    const s = JSON.parse(raw);
    document.getElementById('apiKey').value = s.apiKey || '';
    document.getElementById('baseUrl').value = s.baseUrl || 'https://api.openai.com/v1';
    document.getElementById('model').value = s.model || 'gpt-4o-mini';
    document.getElementById('threshold').value = s.threshold || '0.75';
    document.getElementById('minWords').value = s.minWords || '4';
  } catch (e) {
    console.error('Failed to load settings', e);
  }
}

loadSettings();
```

- [ ] **Step 2: 美化消息列表与状态样式**

在 `<style>` 中添加：

```css
main {
  display: flex;
  flex-direction: column;
}
.message {
  box-shadow: 0 1px 2px rgba(0,0,0,0.05);
  word-wrap: break-word;
}
.assistant {
  border-left: 4px solid #2c3e50;
}
#status {
  font-weight: 500;
}
#status.listening { color: #16a34a; }
#status.speaking { color: #2563eb; }
#status.error { color: #dc2626; }
```

并更新 `updateStatus` 以切换 class：

```javascript
function updateStatus(text) {
  const el = document.getElementById('status');
  el.textContent = text;
  el.className = '';
  if (text.includes('Listening')) el.classList.add('listening');
  if (text.includes('Speaking')) el.classList.add('speaking');
  if (text.includes('Error')) el.classList.add('error');
}
```

- [ ] **Step 3: 在页面加载时显示麦克风权限引导**

如果权限未授予，显示一个覆盖提示：

```html
<div id="permissionHint" style="position:fixed; inset:0; background:rgba(0,0,0,0.6); color:white; display:flex; align-items:center; justify-content:center; z-index:100;">
  <div style="background:#2c3e50; padding:24px; border-radius:8px; text-align:center;">
    <p>Please allow microphone access to start practicing.</p>
    <button id="allowMic">Allow Microphone</button>
  </div>
</div>
```

```javascript
document.getElementById('allowMic').addEventListener('click', async () => {
  try {
    await navigator.mediaDevices.getUserMedia({ audio: true });
    document.getElementById('permissionHint').style.display = 'none';
    startListening();
  } catch (err) {
    alert('Microphone permission is required.');
  }
});
```

- [ ] **Step 4: 在浏览器中测试设置保存与加载**

手动测试：刷新页面后设置仍然保留。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: settings persistence, permission hint, and polished UI"
```

---

## Task 7: 最终集成测试与交付

**Files:**
- Modify: `index.html`（仅修复集成测试中发现的问题）

- [ ] **Step 1: 逐项验证验收标准**

| 验收项 | 验证方式 |
|--------|----------|
| 页面打开后，授权麦克风即可自动开始监听 | 刷新页面，点击 Allow Microphone，状态显示 Listening... |
| 用户说英语，助手以设定人设用英语回应 | 说 "Tell me about your job."，回复应提及 Yutong / product marketing |
| 双方英文文本实时显示在页面上 | 查看对话消息列表 |
| 用户复述/小幅改写助手上一句时，助手不回应并给出提示 | 复述助手最后一句，应看到 shadowing hint |
| 助手朗读时暂停识别，朗读结束后自动恢复监听 | 助手回复时状态为 Speaking...，结束后恢复 Listening... |
| API Key 等基础设置可在页面配置 | 打开 Settings 面板，填写后保存，刷新仍保留 |

- [ ] **Step 2: 修复测试中发现的问题**

根据实际测试结果修改 `index.html`。

- [ ] **Step 3: 最终 Commit（如无修改可跳过）**

```bash
git add index.html
git commit -m "fix: integration test fixes"
```

- [ ] **Step 4: 告知用户可在浏览器中打开 `index.html` 测试**

---

## 自评检查

**Spec 覆盖：**
- 自动监听麦克风：Task 2
- 英文对话与人设：Task 4
- 实时文本显示：Task 2
- 跟读检测静默：Task 3 + Task 4
- 朗读暂停/恢复监听：Task 5
- 设置面板：Task 6

**占位扫描：** 无 TBD/TODO，每步含具体代码或明确手动测试步骤。

**类型一致性：** `isShadowing`、`sendToLLM`、`speak`、`handleUserInput` 签名在各任务中保持一致。
