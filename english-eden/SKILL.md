---
name: english-eden
description: Ashley英语乐园内容生成 - Midjourney生图 + Suno生歌
metadata:
  {
    "openclaw": { "emoji": "📚" }
  }
---

# english-eden

 Ashley英语乐园内容生成工作流

## 概述

从Notion读取"素材收集"状态的任务，生成：
- 🎵 音乐（Suno）
- 🖼️ 图片（Midjourney）

## 前置要求

- Notion API Key 已配置
- 浏览器已登录 Midjourney 和 Suno

## Notion 数据库

- 数据库ID: `23f4578b-9e14-811d-902c-000bf7f3c7d7` (我的发文)
- 筛选条件: `状态 = "素材收集"`

## 工作流程

### 1. 读取任务

```bash
# 查询素材收集状态的任务
NOTION_KEY=$(cat ~/.config/notion/api_key)
curl -s -X POST "https://api.notion.com/v1/data_sources/23f4578b-9e14-811d-902c-000bf7f3c7d7/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {"property": "状态", "select": {"equals": "素材收集"}},
    "page_size": 5
  }' | jq '[.results[] | {id, title: .properties["标题"]?.title[0]?.text?.content}]'
```

### 2. 读取任务详情

获取单条任务的完整信息：
- 标题 (Title)
- 英文标题 (English Title)
- 单词清单
- 提示词-上集 / 提示词-下集
- 英文儿歌 (歌词)
- 编曲风格
- 音频名称

### 3. 音乐生成 (Suno)

**步骤：**
1. 打开 Suno 网站
2. 点击 Create → Custom Mode
3. 填写：
   - Song Name: 音频名称
   - Lyrics: 英文儿歌内容
   - Style: 编曲风格
4. 生成 → 等待完成
5. 下载 MP3 文件
6. 上传到 Notion 的"歌曲下载"字段
7. 删除本地 MP3 文件

**注意：**
- 如果检测到机器人操作，立即关闭浏览器窗口
- 立即通知用户

### 4. 图片生成 (Midjourney)

**步骤：**
1. 打开 Midjourney 网站
2. 在输入框输入提示词
3. 每批输入3条提示词
4. 等待图片生成完成
5. 重复直到全部完成

**分批策略：**
- 每批3条提示词
- 等待5分钟再下一批
- 遇机器人检测 → 关窗口 → 立即通知用户

### 5. 更新 Notion 状态

生成完成后，更新任务状态为下一个阶段（如"内容拼接"）：

```bash
curl -X PATCH "https://api.notion.com/v1/pages/{page_id}" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"properties": {"状态": {"select": {"name": "内容拼接"}}}}'
```

## 通知机制

每次操作都要通知用户：
- ✅ 开始生成音乐
- ✅ 音乐生成成功
- ✅ 开始生成图片（第X批）
- ✅ 图片生成成功
- ⚠️ 遇到机器人检测

## 错误处理

- 机器人检测：关闭浏览器，立即通知用户
- 网络错误：重试3次，失败则通知用户
- API错误：记录日志，通知用户

## 重要经验总结

### ✅ 已验证可行
- Notion API 可以读取数据库和更新状态
- 浏览器自动化可以：点击、输入文本、导航、截图
- Suno 生成音乐成功
- Midjourney 生成图片成功

### ⚠️ 已知限制
- **浏览器无法处理文件上传对话框**（操作系统原生对话框，自动化工具无法控制）
- 解决方案：
  1. 手动上传文件到 Notion
  2. 使用 URL 链接（Notion 支持 URL 类型字段）
  3. 使用 Suno 生成的歌曲链接直接填入

### 📋 实际操作流程

**每次任务流程：**
1. 从 Notion 读取"素材收集"状态的任务
2. 生成音乐（Suno）→ 下载到本地
3. 手动上传 MP3 到 Notion 或使用 URL
4. 生成图片（Midjourney）→ 每批3条，间隔5分钟
5. 更新 Notion 状态为"内容拼接"

---

## 示例任务数据

典型"素材收集"状态的任务包含字段：
- 标题: "🦋 树栖昆虫 ｜英语启蒙快乐记单词"
- 单词清单: "1. Leaf Bug /liːf bʌɡ/ 树叶虫 ..."
- 提示词-上集: "A 3D cartoon Leaf Bug with plush green texture..."
- 提示词-下集: "A 3D cartoon Cicada with plush brown texture..."
- 英文儿歌: "This is a Leaf Bug, Leaf Bug. This is a Treehopper..."
- 编曲风格: "A bright and playful children's song with buzzing sound effects..."
- 音频名称: 可用于Suno的歌曲名称
