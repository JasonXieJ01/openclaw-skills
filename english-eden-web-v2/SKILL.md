---
name: english-eden-web-v2
description: Ashley英语乐园内容生成（Web版稳健流程）。Use when tasks come from Notion with 状态=素材收集 and execution should automate Suno web music generation + Midjourney web image generation, then write back song file/link and update status in Notion with retries, batching, and anti-bot handling.
---

# english-eden-web-v2

执行 Notion → Suno(Web) → Midjourney(Web) 的稳健流水线。

## Inputs

- Notion 数据源 ID（默认：`23f4578b-9e14-811d-902c-000bf7f3c7d7`）
- 筛选状态（默认：`素材收集`）
- 每次处理条数（默认：1，最大 3）

## Required fields in Notion

- 标题
- 英文儿歌（歌词）
- 编曲风格
- 音频名称
- 提示词-上集
- 提示词-下集
- 状态

## Execution policy

- 全程自动执行，不要求用户手工介入。
- 任何网站出现机器人校验：立即停止当前站点操作、截图留证、通知失败原因。
- 网络/API错误最多重试 3 次（指数退避：5s/15s/30s）。

## Workflow

1. 查询 Notion 中 `状态=素材收集` 的任务（按创建时间升序，避免跳单）。
2. 读取单条任务全部字段并做字段校验；缺字段则写回“阻塞原因”并跳过。
3. 在 Suno Web 执行生成：
   - Create → Custom
   - Song Name=音频名称
   - Lyrics=英文儿歌
   - Style=编曲风格
   - 提交并等待完成（轮询上限 20 分钟）
4. 获取 Suno 成品：
   - 优先拿可公开下载 URL
   - 如有本地文件，上传到 Notion File Upload API，获得 `file_upload.id`
   - 回填 Notion “歌曲下载”字段（files）
5. 在 Midjourney Web 执行图片生成：
   - 合并提示词（上集+下集）
   - 每批 3 条 prompt，批次间固定等待 5 分钟
   - 每条完成后记录结果 URL 到任务备注/结果字段
6. 全部成功后将状态更新为 `内容拼接`。

## Notion file write-back (自动上传优先)

优先方案：Notion File Upload API（自动上传）。

- 步骤：create upload → send binary → complete upload → page property attach
- 若 API/权限不支持二进制上传，则降级为 external URL 写入 files 字段。

示例（external URL 降级写法）：

```bash
curl -s -X PATCH "https://api.notion.com/v1/pages/$PAGE_ID" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "歌曲下载": {
        "files": [
          {
            "name": "song.mp3",
            "external": {"url": "https://example.com/song.mp3"}
          }
        ]
      }
    }
  }'
```

## Midjourney Web anti-flake rules

- 使用稳定 selector（优先 aria/role）。
- 每一步操作后校验页面状态再继续。
- 批次内失败 1 条不影响后续条目，但任务最终标记为“部分完成”。

## Notifications

在关键节点发通知：

- 开始处理任务
- Suno 生成成功/失败
- Notion 文件回填成功/失败
- Midjourney 第 N 批开始/完成
- 任务最终状态（成功/部分完成/失败）

## Output contract

每条任务输出：

- `task_id`
- `music_status`（success/failed）
- `music_asset`（file_upload_id 或 external_url）
- `image_status`（success/partial/failed）
- `image_urls[]`
- `final_notion_status`
- `error_summary`（如有）
