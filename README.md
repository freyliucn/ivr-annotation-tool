# Riyadh Bank IVR 测试标注平台

## 启动方式

翻译功能需要 HTTP 服务器（浏览器 CORS 限制，不能直接双击 `file://` 打开）：

```bash
cd annotation
./start.sh
# 或手动：
python3 -m http.server 8080
# 浏览器访问 http://localhost:8080
```

## 功能现状

### ✅ 已完成
| 功能 | 说明 |
|------|------|
| 测试用例浏览 | 左侧列表展示 test_id / category / language，支持点击选中 |
| 筛选搜索 | 按 Category、Language 下拉筛选，支持关键词搜索 |
| 键盘导航 | ↑↓ 方向键 / j/k 键切换上下条用例 |
| 剧本化对话展示 | 聊天气泡 UI，User 左侧 / Agent 右侧 |
| 双行显示 | 上行：原文（阿语/英语），下行：中文翻译（灰色小字） |
| 原文直接编辑 | contenteditable，可修正不标准用词 |
| 中文翻译编辑 | 可修改翻译大意 |
| LLM 翻译 | 点击「🔄 翻译」调用 NVIDIA Qwen3.5 免费模型生成单句中文翻译 |
| 批量全篇翻译 | 点击「🔄 翻译全篇」根据设置的并发数快速翻译对话中所有未翻译消息 |
| 反向重生成 | 修改中文大意后点「🔄 重生成」，LLM 根据中文重新生成原文 |
| 期望行为编辑 | action 类型选择 + language/branch_id/intent_id 参数编辑 |
| 分行搜索器 | 可搜索 247 个分行（支持编号、英文名、阿拉伯文名） |
| 意图选择器 | 8 个意图下拉（0-7） |
| 新建/删除用例 | 创建空白用例 / 删除已有用例 |
| 添加/删除消息 | 在对话中添加 Customer/Agent 消息轮次 |
| 导入 JSONL | 加载 `agent_regression_tests.jsonl` 格式文件 |
| 导出 JSONL | 导出为评估管线兼容的 JSONL |
| 导出 Markdown | 导出为人类可读的剧本化 test_scripts.md |
| 覆盖率仪表板 | 分行覆盖率、分类/意图/语言分布图表 |
| 设置面板 | 配置 API Base URL / API Key / 翻译模型 |
| 左侧列表滚动功能自动定位 | 引入独立滚动上下文，支持键盘 ↑↓ 切换时自动滚动到可视区域 |

### 🐛 已知问题

暂无

## 技术栈

- **前端**: 单 HTML 文件，Vue 3 (CDN) + 内联 CSS
- **字体**: JetBrains Mono（代码）、Noto Sans SC（中文）、Noto Sans Arabic（阿拉伯文）
- **翻译 API**: NVIDIA NIM — `qwen/qwen3.5-122b-a10b`（免费）
- **数据存储**: 浏览器 localStorage
- **设计风格**: Industrial/Utilitarian 深色主题

## 翻译服务配置

| 配置项 | 默认值 |
|--------|--------|
| API Base | `/api/chat`（推荐） |
| Model | `qwen/qwen3.5-122b-a10b` |
| API Key | 直连模式需要；代理模式可留空 |

支持切换到其他 OpenAI 兼容 API（如 OpenRouter、DashScope），在设置面板修改即可。

### 网络排障（翻译出现 Network error）

优先使用代理模式（`API Base = /api/chat`）：

1. **本地运行**
   - 启动：`./start.sh`
   - 若提示 `Cannot reach proxy endpoint (/api/chat)`，说明本地服务未启动或已退出，重启 `./start.sh` 即可。

2. **Vercel 部署**
   - 确认已部署 `api/chat.js`
   - 确认项目环境变量里有 `NVIDIA_API_KEY`（并触发了重新部署）
   - 若提示 proxy non-JSON 或 5xx，请查看 Vercel Function 日志

3. **不推荐直连上游**
   - 直连 `https://integrate.api.nvidia.com/v1/chat/completions` 可能受浏览器 CORS 限制
   - 建议统一走 `/api/chat` 代理，避免前端网络/CORS不确定性

## 数据格式

### 输入/输出 JSONL 格式
```json
{
  "test_id": "TC_HP_001",
  "category": "HappyPath",
  "description": "Full slot extraction EN: Al-Ola/CSR",
  "context": { "language": "English", "simulated_asr_confidence": 0.96 },
  "conversation": [
    { "role": "user", "content": "Hi, I need to speak with CSR at the Al-Ola branch." }
  ],
  "expected_behavior": {
    "action": "call_tool:transfer_call",
    "parameters": { "language": "English", "branch_id": "414", "intent_id": "1" }
  },
  "eval_metrics": {
    "must_trigger_action": "call_tool:transfer_call",
    "exact_match_params": ["language", "branch_id", "intent_id"]
  }
}
```

### 分类定义
| Category | 说明 |
|----------|------|
| HappyPath | 完整槽位，一次转接 |
| MissingSlots | 缺失槽位，需反问 |
| Ambiguity | 近音/歧义分行 |
| NoiseFallback | 闲聊/噪音/无意图 |
| NegationCorrection | 否定修正 |
| AdversarialParams | 对抗性参数 |

### 意图定义
| ID | 名称 |
|----|------|
| 0 | Operator / 人工座席 |
| 1 | CSR / 客服 |
| 2 | Teller / 柜台 |
| 3 | Diamond / 钻石理财 |
| 4 | Golden / 黄金理财 |
| 5 | Corporate / 企业银行 |
| 6 | Riyadh Capital / 利雅得资本 |
| 7 | Manager / 经理 |

## 文件结构

```
annotation/
├── index.html      # 标注平台主文件（单文件应用）
├── start.sh        # 启动脚本（python3 HTTP server）
└── README.md       # 本文档
```
