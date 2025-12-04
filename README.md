# 🎙️ AI Assistant - 语音聊天助手

一个端到端的语音聊天助手/陪伴机器人项目，提供自然的语音交互体验。

## ✨ 功能特性

- 🎤 **语音输入** - 实时语音识别，支持中文对话
- 🤖 **智能对话** - 基于大语言模型的智能回复
- 🔊 **语音输出** - 自然语音合成，流畅的对话体验
- 📝 **文本处理** - 支持摘要、总结等文本任务

## 🚀 快速开始

### 环境要求

- Python 3.11+
- [UV](https://github.com/astral-sh/uv) - 现代化的 Python 包管理器

### 安装 UV

```bash
# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 项目设置

```bash
# 克隆项目
git clone <repository-url>
cd ai-assistant

# 创建虚拟环境并安装依赖
uv venv
uv sync

# 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入你的 API 密钥
```

### 运行项目

```bash
# 激活虚拟环境 (可选，uv run 会自动使用)
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate

# 运行主程序
uv run python src/main.py
```

## 📁 项目结构

```
ai-assistant/
├── AGENTS.md           # AI Agent 指导文件
├── CLAUDE.md           # Claude 专用配置
├── README.md           # 项目说明文档 (本文件)
├── .env.example        # 环境变量模板
├── .env                # 实际环境变量 (需自行创建)
├── .gitignore          # Git 忽略配置
├── pyproject.toml      # 项目配置和依赖
├── src/                # 源代码目录
│   ├── __init__.py
│   ├── main.py         # 主入口
│   ├── llm/            # LLM 相关模块
│   ├── voice/          # 语音处理模块
│   └── utils/          # 工具函数
├── tests/              # 测试目录
└── docs/               # 文档目录
```

## ⚙️ 配置说明

### 环境变量

复制 `.env.example` 为 `.env` 并配置以下变量：

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `LLM_API_KEY` | 智谱 AI API 密钥 | `your-api-key` |
| `EMBEDDING_API_KEY` | Gitee AI Embedding 密钥 | `your-api-key` |
| `RERANK_API_KEY` | Gitee AI Rerank 密钥 | `your-api-key` |

### 模型配置

| 用途 | 模型 | 说明 |
|------|------|------|
| 一般对话 | `glm-4.5-flash` | 快速响应，适合实时对话 |
| 摘要总结 | `glm-4-flash-250414` | 适合文本处理任务 |
| 文本嵌入 | `bge-m3` | 1024 维向量 |
| 重排序 | `bge-reranker-v2-m3` | 语义相关性排序 |

## 🧪 开发指南

### 代码质量

```bash
# 格式化代码
uv run ruff format .

# 检查代码
uv run ruff check .

# 运行测试
uv run pytest

# 查看测试覆盖率
uv run pytest --cov=src
```

### 添加依赖

```bash
# 安装单个包
uv pip install <package-name>

# 同步所有依赖
uv sync
```

## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m '✨ feat: 添加某功能'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建 Pull Request
