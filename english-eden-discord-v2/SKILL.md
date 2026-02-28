---
name: english-eden-discord-v2
description: Ashley英语乐园内容生成（Discord版 Midjourney）。Use when tasks come from Notion with 状态=素材收集 and execution should automate Suno web music generation plus Midjourney Discord /imagine workflow, then write back song file/link and update task status in Notion.
---

# english-eden-discord-v2

执行 Notion → Suno(Web) → Midjourney(Discord Bot) 的可切换替代流程。

## Inputs

- Notion 数据源 ID（默认：`23f4578b-9e14-811d-902c-000bf7f3c7d7`）
- Discord 目标频道（Midjourney 可用频道）
- 筛选状态（默认：`素材收集`）

## Required fields in Notion

- 标题
- 英文儿歌（歌词）
- 编曲风格
- 音频名称
- 提示词-上集
- 提示词-下集
- 状态

## Workflow

1. 从 Notion 拉取待处理任务（状态=素材收集）。
2. Suno Web 生成音乐并等待完成。
3. 回填 Notion “歌曲下载”：
   - 优先 File Upload API 自动上传
   - 失败则 external URL 降级
4. Midjourney 走 Discord Bot：
   - 将 prompt 分批（每批 3 条）
   - 在 Discord 频道发送 `/imagine`（或等价 bot 交互）
   - 监听返回消息中的图片链接
   - 记录每条 prompt 的结果 URL
5. 更新 Notion 状态为 `内容拼接`（或部分完成时写入失败原因）。

## Discord route advantages

- 消息驱动，状态可观测性高
- 批量与重试更容易标准化
- 相比 Web UI，DOM 变动影响更小

## Batch and retry policy

- 每批 3 条，批次间等待 5 分钟
- 单条失败重试 2 次
- 整体失败率 >50% 时停止任务并标记失败

## Notion write-back example (status)

```bash
curl -s -X PATCH "https://api.notion.com/v1/pages/$PAGE_ID" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"properties":{"状态":{"select":{"name":"内容拼接"}}}}'
```

## Notifications

关键通知：

- 任务开始
- Suno 成功/失败
- Midjourney 第 N 批开始/完成
- Notion 状态更新结果
- 最终总结

## Output contract

每条任务输出：

- `task_id`
- `music_status`
- `music_asset`
- `mj_mode`（discord）
- `image_urls[]`
- `success_count/failed_count`
- `final_notion_status`
- `error_summary`
