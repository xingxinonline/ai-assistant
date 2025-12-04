# AGENTS.md - AI Assistant 语音聊天项目

## 项目概述
这是一个端到端的语音聊天助手/陪伴机器人项目，支持语音输入、智能对话和语音输出。

## 开发环境

### 环境管理
- 使用 **UV** 管理 Python 虚拟环境和依赖
- Python 版本: 3.11+
- 激活环境: `uv venv && .venv\Scripts\activate` (Windows)
- 安装依赖: `uv pip install -r requirements.txt` 或 `uv sync`
- 运行脚本: `uv run python <script.py>`

### 常用命令
- 创建虚拟环境: `uv venv`
- 安装依赖: `uv pip install <package>`
- 运行测试: `uv run pytest`
- 代码格式化: `uv run ruff format .`
- 代码检查: `uv run ruff check .`

## API 配置

### LLM 服务
- 一般对话使用 `glm-4.5-flash` 模型
- 摘要/总结使用 `glm-4-flash-250414` 模型
- API Host: `https://open.bigmodel.cn/api/paas/v4`

### Embedding 服务
- 模型: `bge-m3`
- 维度: 1024
- API Host: `https://ai.gitee.com/v1`

### Rerank 服务
- 模型: `bge-reranker-v2-m3`
- API Host: `https://ai.gitee.com/v1/rerank`

## 项目结构
```
ai-assistant/
├── AGENTS.md           # AI Agent 指导文件
├── CLAUDE.md           # Claude 专用配置
├── README.md           # 项目说明文档
├── .env.example        # 环境变量模板
├── .env                # 实际环境变量 (不提交到 Git)
├── .gitignore          # Git 忽略配置
├── pyproject.toml      # 项目配置和依赖
├── src/                # 源代码目录
│   ├── __init__.py
│   ├── llm/            # LLM 相关模块
│   ├── voice/          # 语音处理模块
│   └── utils/          # 工具函数
├── tests/              # 测试目录
└── docs/               # 文档目录
```

## 测试指令
- 运行全部测试: `uv run pytest`
- 运行指定测试: `uv run pytest tests/test_<name>.py -v`
- 查看覆盖率: `uv run pytest --cov=src`

## 代码规范
- 使用 Ruff 进行代码格式化和检查
- 所有函数需要类型注解
- 遵循 PEP 8 风格指南
- 使用中文编写文档字符串

## PR 指令
- 标题格式: `[模块名] 简要描述`
- 提交前运行: `uv run ruff check . && uv run pytest`
- 确保所有测试通过
