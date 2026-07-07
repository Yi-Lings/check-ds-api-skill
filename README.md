# DeepSeek API Usage Checker

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

通过浏览器自动化登录 [DeepSeek 开放平台](https://platform.deepseek.com/usage)，提取用量信息（余额、消费、各模型 Token 消耗、缓存命中率）并保存为结构化 Excel 报告。

> **本仓库是一个 Agent Skill** — 供 [Claude Code](https://claude.ai/code) 等 AI 助手读取并执行的指令文件，而非传统命令行工具。

---

## 安装

将本 skill 添加到你的 Claude/Codex/Openclaw 项目中：
1.对智能体直接说：
```bash
帮我安装这个skill "https://github.com/Yi-Lings/check-ds-api-skill/"
```
2.手动安装：
```bash
# 进入你的项目根目录
cd /path/to/your/project

# 创建 skills 目录（如不存在）
mkdir -p .claude/skills

# 克隆本仓库到 skills 目录
git clone https://github.com/Yi-Lings/check-ds-api-skill.git .claude/skills/check_ds_api
```

### 依赖

| 依赖 | 安装 |
|------|------|
| Claude/Codex/Openclaw| 依赖playwright-cli |
| Python ≥ 3.8 + openpyxl | `pip install openpyxl` |

> **Windows 注意：** 必须使用 `python`（而非 `python3`）命令。

---

## 用法

在 Agent 中直接说：

```
查 DeepSeek 用量
看 DeepSeek 余额
查看 platform.deepseek.com 使用量
查 API 消耗
```

AI 助手会按 `SKILL.md` 的步骤自动执行完整流程。

---

## 工作原理

1. 通过 Playwright 打开 DeepSeek 开放平台用量页面
2. 拦截后端 API 请求获取真实数据（UI 不展示输入/输出拆分和缓存命中率）
3. 调用 Python 脚本生成包含 4 个工作表的 Excel 报告

---

## License

[MIT](LICENSE)
