---
type: learning_concept
status: practiced
created: 2026-11-03
updated: 2026-11-03
domain: obsidian
stack: 工具
source: 插件配置记录
level: beginner
prerequisites: []
related: [Obsidian操作指南]
tags:
  - learn/concept
---

# Obsidian 插件简介

## 0. 一句话理解
本库启用的 5 个社区插件及其用途，配置于 2026-11-03，与源知识库保持一致。

## 1. 插件清单

| 插件 | 简介 |
| --- | --- |
| Copilot | 在库内运行 AI 助手（如 Claude Code、Codex、OpenCode），把第二大脑打造成智能助理 |
| Excalidraw | 思维草图与可视化笔记，支持编辑和查看 Excalidraw 绘图 |
| Git | 集成 Git 版本控制，支持自动备份等高级功能 |
| Omnisearch | 全局搜索引擎，快速精准地查找笔记内容 |
| Templater | 高级模板和自动化工具，支持 Handlebars 风格语法 |

## 2. Copilot 配置要点

- 接入 DeepSeek 第三方大模型 API（OpenAI 兼容接口，`https://api.deepseek.com`），聊天默认模型 deepseek-flash / deepseek-v4-pro。
- 另有 OpenCode 免费模型（Big Pickle 等）可用。
- API Key 存于 Obsidian 应用级加密钥匙串（DPAPI），不在笔记与配置文件中明文保存。
- 对话记录、自定义提示词、AI 记忆存放在 `90_附件/插件库/copilot/`。
- 使用前提：FlClash 等代理保持运行（系统代理或 TUN 模式），否则国内网络无法访问 API。

## 3. 其他插件要点

- **Excalidraw**：图纸、模板、脚本路径配置在 `90_附件/插件库/Excalidraw/`。
- **Git**：配合 GitHub 私有仓库备份；自动提交/推送需在插件设置中开启，且依赖代理连通 GitHub。
- **Templater**：当前使用默认配置；核心模板插件用 Obsidian 自带 Templates（模板目录 `00_知识库管理/模板`）。
- **Omnisearch**：默认配置即可用。

## 4. 使用提醒

- 插件配置随 `.obsidian/` 目录一起纳入 git 版本管理（`workspace.json` 除外）。
- 换新设备克隆仓库后，插件会自动安装同版本；Copilot 的 API Key 需在新设备上重新输入一次（钥匙串加密数据不跨设备）。

## 5. 参考来源
- 源知识库《Obsidian插件简介》（2026-11-03 迁入整理）
- 相关笔记：[[Obsidian操作指南]]、[[00_知识库入口]]
