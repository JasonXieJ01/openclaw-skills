# OpenClaw Skills

OpenClaw 自动化技能集合

## 当前技能

### 📚 english-eden
Ashley英语乐园内容生成 - Midjourney生图 + Suno生歌

从Notion读取"素材收集"状态的任务，自动生成：
- 🎵 音乐（Suno）
- 🖼️ 图片（Midjourney）

详见 [english-eden/SKILL.md](./english-eden/SKILL.md)

### 📚 english-eden-web-v2
稳健版 Web 流程（Suno Web + Midjourney Web），含重试、分批、异常处理和 Notion 自动回填策略。

详见 [english-eden-web-v2/SKILL.md](./english-eden-web-v2/SKILL.md)

### 📚 english-eden-discord-v2
可切换 Discord 版流程（Suno Web + Midjourney Discord Bot），更适合批量和稳定追踪。

详见 [english-eden-discord-v2/SKILL.md](./english-eden-discord-v2/SKILL.md)

### 📚 english-eden-discord-v3
状态机版流程（严格阶段开始/完成汇报、单主页执行、下载能力分流、失败可审计）。

详见 [english-eden-discord-v3/SKILL.md](./english-eden-discord-v3/SKILL.md)

---

## 如何使用

1. 克隆此仓库到本地
2. 将 skill 文件复制到 `~/.npm-global/lib/node_modules/openclaw/skills/`
3. 重启 OpenClaw
