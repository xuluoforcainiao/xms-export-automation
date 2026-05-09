---
name: xms-export-automation
description: Automates the XMS customer service abnormal item export workflow. Supports per-webhook scheduling so different DingTalk groups can receive exports at different frequencies. Covers SSO login, clearing the shelf organization filter, exporting abnormal items, polling the download center, uploading to a specified DingTalk document folder, and sending permanent links via DingTalk custom robots. Use when the user needs to export XMS abnormal items, set up scheduled XMS exports, configure per-group delivery schedules, or manage DingTalk delivery for XMS data.
---

# XMS 异常件导出自动化

## Overview

This skill automates the full workflow of exporting abnormal items from the XMS customer service system and delivering them to DingTalk groups via permanent document links. Each DingTalk group (webhook) can have its own independent schedule.

## Workflow Steps

### Phase 0: Check Which Webhooks Should Run
1. Read `xms_export_config.json`
2. Get current Beijing time and current day of week (0=Sunday, 1=Monday, ..., 6=Saturday)
3. For each webhook:
   - Check `enabled` is true
   - Check `schedule.timeSlots` for a match with current hour/minute
   - Check `schedule.days` contains current day of week
4. Collect all matching webhooks. If none match, exit immediately without any action
5. Only matching webhooks will receive messages in Phase 5

### Phase 1: Login
1. Navigate to `http://cs.packet.i4px.com/`
2. If redirected to SSO (`sso.i4px.com`), log in with configured credentials
3. Wait for redirect to `cs-packet.i4px.com/index`

### Phase 2: Export Data
1. Click "异常件管理" in the left sidebar to expand submenu
2. Click "自有业务异常件管理"
3. **Critical**: Clear the "上架组织" field. The field is a text input containing "客户服务部". Do NOT click on the "上架组织" label itself — it will only select the label text. Instead:
   - Left-click **inside the input field**, on the right side of the "客户服务部" text (avoid the left-side label area)
   - Press Ctrl+A to select all text in the input
   - Press Delete to clear it
   - The field should now show the placeholder "双击选择上架组织"
   - If still not empty, retry up to 3 times. The field must be empty before querying.
4. Click "查询"
5. Wait for data to load, then click "导出原因（新）" in the top-right

### Phase 3: Poll for Completion
1. Click "下载中心" (or "导出记录") to open the export records dialog
2. Check the progress column of the latest (first) record
3. If not 100%, wait 2 minutes and refresh to recheck
4. Repeat until 100% (max wait: 70 minutes)

### Phase 4: Read Creation Time and Download
1. Read the "创建时间" column text from the latest record (format: `2026-05-09 07:03:15`)
2. Name the file: `xms异常件导出_{timestamp}.xlsx` (e.g., `xms异常件导出_2026-05-09 07:03:15.xlsx`)
3. Click download and wait for completion
4. Rename the downloaded file to match the creation timestamp

### Phase 5: Upload to DingTalk and Send
1. Call `get_file_upload_info` to get upload credentials
2. Upload the xlsx file via HTTP PUT to the returned `resourceUrl`
   - Set `Content-Type` header to empty string
   - Include returned `Authorization` and `x-oss-date` headers
3. Call `commit_uploaded_file` with:
   - `folderId`: The target DingTalk folder ID (configured in settings)
   - `name`: The creation-timestamp-based filename from Phase 4
   - Record the returned `nodeId`
4. Generate permanent link: `https://alidocs.dingtalk.com/i/nodes/{nodeId}`
5. **Do NOT** use `download_file` — it returns temporary OSS links that expire in ~15 minutes
6. Send messages **only to webhooks matched in Phase 0** via `send_message_by_custom_robot`
   - Each message must include that webhook's configured keyword
   - Include the permanent document link

## Configuration File

Read from `xms_export_config.json`:

```json
{
  "xms": {
    "url": "http://cs.packet.i4px.com/",
    "username": "...",
    "password": "..."
  },
  "webhooks": [
    {
      "name": "Group A",
      "token": "access_token_value",
      "keyword": "required_keyword_in_message",
      "enabled": true,
      "schedule": {
        "timeSlots": [{"hour": 9, "minute": 0}],
        "days": [1, 2, 3, 4, 5]
      }
    },
    {
      "name": "Group B",
      "token": "access_token_value",
      "keyword": "required_keyword_in_message",
      "enabled": true,
      "schedule": {
        "timeSlots": [{"hour": 17, "minute": 0}],
        "days": [1, 2, 3, 4, 5, 6, 0]
      }
    }
  ],
  "export": {
    "poll_interval_seconds": 120,
    "max_wait_minutes": 70,
    "file_name_template": "xms异常件导出_{timestamp}.xlsx"
  }
}
```

### Cron Expression

The QoderWork cron job should be the union of all webhook time slots:

```
0 9,17 * * *
```

This means the task fires at 09:00 and 17:00 every day. Phase 0 then filters which webhooks actually match the current time.

## Critical Rules

- **Per-webhook scheduling**: Each webhook has its own `schedule.timeSlots` and `schedule.days`. The task only sends to webhooks whose schedule matches the current time
- **Filename**: Must use the "创建时间" from the export record, never system time or hardcoded dates
- **上架组织**: Must be empty before querying. Click inside the input field (right side of the text, avoiding the label), then Ctrl+A, then Delete to clear "客户服务部"
- **DingTalk links**: Always use `https://alidocs.dingtalk.com/i/nodes/{nodeId}` format, never temporary OSS download links
- **Keywords**: Every DingTalk robot message must contain that webhook's configured keyword or it will be rejected
- **Folder ID**: `commit_uploaded_file` must include `folderId` to place files in the correct DingTalk folder
- **Model 配置（强制标准模型）**: 创建 QoderWork 定时任务时，`payload.model` 必须显式设置为 `"qwork-auto"`（标准模型）。禁止使用旗舰模型（`qwork-ultimate`），以避免不必要的 credit 消耗。如果用户未指定模型，默认强制使用 `qwork-auto`
