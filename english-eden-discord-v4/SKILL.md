---
name: english-eden-discord-v4
description: Ashley英语乐园全流程生产（Notion + Suno + Midjourney Discord/Web）状态机+防停机版本。Use when tasks come from Notion with 状态=素材收集 and you need strict phase-based execution, anti-stall watchdog behavior, proactive progress reports, single-channel continuous run, robust browser action retries (snapshot+act fallback), 3-item batching, dynamic video pacing, audio normalization, and deterministic download fallback.
---

# english-eden-discord-v4

把“可跑通”升级为“可控、可恢复、可审计”。

**v4 强化目标：** 单频道长流程持续执行，优先解决“阶段边界停住 / 浏览器动作参数不完整 / 只汇报不执行”。

## Hard rules

## Hard Rule 0 — Repair mode (default ON)

When execution stalls or repeats without visible progress, enter Repair Mode automatically.

Repair Mode steps:
1. Report latest tool error verbatim (time + error text).
2. If error contains `browser failed: fields are required`, MUST switch to:
   - `browser.snapshot(refs=aria)`
   - then `browser.act` with complete `fields` payload (targetId/ref/fields explicit)
   - avoid empty/incomplete browser calls
3. For every action, output: `动作名 + 参数摘要 + 结果(成功/失败)`
4. If same error repeats 2 times, stop retry loop and emit:
   - `[阶段X 等待人工] <唯一一步人工操作>`
5. Before phase5, forbid graceful exit text such as “已完成/我现在继续执行” without a real next action.

Stall detector:
- No browser-visible progress for 120s OR same status text repeated 2 times => trigger Repair Mode immediately.


1. 强制状态机执行：每阶段只有 `pending -> running -> done/failed`。
2. 强制进度播报：每阶段必须发送两条消息：
   - `[阶段X 开始] ...`
   - `[阶段X 完成] ...`
3. 超过 10 分钟无阶段切换，发送“进行中”心跳。
4. 浏览器操作默认单主页面；临时子页用完即关。
5. 任何阶段失败必须写入 Notion `生成日志` 并标记失败原因。

## Phase plan

- 阶段1：Notion取数与字段校验
- 阶段2：Suno生成与音频回填
- 阶段3：MJ图片生成（3条/组）
- 阶段4：MJ视频生成（3条/组，默认5分钟，支持动态提前）
- 阶段5：下载归档与最终交付

## Phase contracts (input/output + transition)

Use the following contract to force continuity and avoid stopping at phase boundaries.

### Phase 1 contract
- Input: Notion DB query filter `状态=素材收集`
- Output (required): `task_page_id`, `title`, `audio_name`, `prompt_count`, `prompts[]`
- Done criteria: required fields complete and `prompt_count==12`
- On done: immediately start Phase 2 (no user confirmation needed)

### Phase 2 contract
- Input: `title`, `audio_name`, lyrics/style fields
- Output (required): normalized audio file path (`<音频名称>.<ext>`), Notion `歌曲(files)` attached id/url
- Done criteria: file extension valid + upload attached in Notion
- On done: immediately start Phase 3

### Phase 3 contract
- Input: `prompts[]` (12)
- Output (required): image job records for 12 prompts
- Done criteria: every prompt has image result record
- On done: immediately start Phase 4

### Phase 4 contract
- Input: image job records
- Output (required): video job records for 12 prompts
- Done criteria: every prompt has 1 selected image + 1 video result
- On done: immediately start Phase 5

### Phase 5 contract
- Input: video/audio artifacts
- Output (required): `output/<音频名称>/` final archive manifest
- Done criteria: 12 videos + 1 audio OR explicit `download_blocked` checklist
- On done: emit final delivery message once

## Runtime message policy (fixed templates)

Use these exact status types only:
- `[阶段X 开始] ...`
- `[阶段X 进行中] ...` (heartbeat every 10 min without phase switch)
- `[阶段X 重试#N] <错误>，将在 <delay>s 后重试`
- `[阶段X 等待人工] <你需要做什么>；完成后回复：继续阶段X`
- `[阶段X 失败] <原因>；已写入 Notion 生成日志`
- `[阶段X 完成] <关键产物>`

Do not send generic “已处理/继续中” without phase tag.

## Batch loop policy (Phase 3/4)

For prompts[1..12], execute in fixed batches of 3:
1. Submit items `(1-3)`, wait/check completion, mark batch done.
2. Submit `(4-6)`, then `(7-9)`, then `(10-12)` with same logic.
3. After each batch done, emit one progress message with counts.

Pseudo-flow:
```text
for batch in chunk(prompts, 3):
  for item in batch:
    run item action
    if retryable_error: retry(3, [30,60,120])
    if human_required: pause
  wait until batch done (or timeout->retry/fail)
  emit progress (done/total)
```


## Visibility and live-progress policy

Before any browser-visible action, emit:
- `[阶段X 进行中] 即将操作页面：<site/page>，动作：<action>`

If no browser action occurs within 60s after a phase starts, emit one diagnostic status:
- `[阶段X 进行中] 当前在执行非页面步骤：<notion/io/wait/retry>`

If blocked longer than 90s, emit blocked reason explicitly:
- `[阶段X 进行中] 页面未变化原因：<队列等待/风控限制/等待人工/隔离浏览器执行>`

Browser targeting rule:
1. 默认优先操作当前已连接的同一浏览器标签页（若可用）。
2. 若切换到隔离浏览器执行，必须先提示：`[阶段X 进行中] 将在隔离浏览器执行，当前窗口不会实时变化`。
3. 执行结束后，输出本阶段实际操作目标（主标签/隔离浏览器）。

## Timeout and recovery

- Single action soft-timeout: 90s; hard-timeout: 180s.
- Batch wait timeout: 8min (Phase3), 10min (Phase4).
- Timeout handling: retry up to 3; on exhaust -> `failed`.
- If run is externally interrupted/aborted, resume from last phase checkpoint in Notion `生成日志` and continue.

## Phase 1 — Notion intake

执行：
1. 查询 `状态=素材收集`。
2. 按创建顺序选择下一条。
3. 校验必填字段：标题、英文儿歌、编曲风格、音频名称、提示词-上集、提示词-下集。
4. 缺字段：写日志并停在当前阶段。

输出：
- `task_page_id`
- `title`
- `audio_name`
- `prompt_count`

## Phase 2 — Suno + audio normalization

执行：
1. Suno生成音乐（Song Name/Lyrics/Style）。
2. 下载音频到临时文件。
3. 检测真实格式（mp3/wav/m4a）。
4. 重命名为 `<音频名称>.<ext>`。
5. 上传到 Notion `歌曲(files)`。
6. 可选回填 `歌曲下载(url)` 作为备用。

规则：
- 禁止无后缀文件作为主交付。
- Notion上传采用：create upload -> upload binary -> attach file_upload.id。

## Phase 3 — MJ image generation

执行：
1. 合并上集+下集提示词（总 12 条）。
2. 每 3 条一组提交。
3. 组间等待固定时间（默认 5 分钟，可配置）。
4. 每组完成后播报一次。

## Phase 4 — MJ video generation

执行：
1. 每个提示词从 4 张图中选 1 张“动作最自然”。
2. 触发 High Motion 动画。
3. 每组 3 条视频。
4. 默认 5 分钟一轮；若队列数字满足“可提前”条件（如剩余2），可提前发起下一条。

规则：
- 单主页面滚动执行。
- 需要详情操作时可开临时子页，动作后立即关闭。

## Phase 5 — Download + archive

目标产物：
- 12 个视频（每提示词1个）
- 1 个音频
- 统一目录：`output/<音频名称>/`

下载策略（按优先级）：
1. 自动下载可用：直接下载并命名归档。
2. 自动下载不可用：进入 `download_blocked` 分支，输出“待下载清单 + 命名模板”，不伪造完成。

命名建议：
- `01-leaf-bug.mp4`
- `02-treehopper.mp4`
- ...
- `12-caterpillar.mp4`
- `<音频名称>.mp3`

## Failure policy

任一阶段失败时：
1. 立即播报 `[阶段X 失败]`。
2. 写入 Notion `生成日志`（时间、步骤、原因、下一步建议）。
3. 不跳过失败继续“伪完成”。

## Minimal progress message template

- `[阶段1 开始] Notion取数与字段校验`
- `[阶段1 完成] 已锁定任务：<title>，共<12>条提示词`
- `[阶段2 开始] Suno生成与音频回填`
- `[阶段2 完成] 音频已回填：<file/url>`
- ...
- `[阶段5 完成] 已归档：<folder>（12视频+1音频）`
