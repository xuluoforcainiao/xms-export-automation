---
name: xms-export-automation
description: Automates the XMS customer service abnormal item export workflow. Covers SSO login, clearing the shelf organization filter, exporting abnormal items, polling the download center, downloading the result file, and optionally uploading to DingTalk. Use when the user needs to export XMS abnormal items, set up scheduled XMS exports, or manage XMS data delivery.
---

# XMS 异常件导出自动化

## Overview

Automates the full workflow of exporting abnormal items from the XMS customer service system (`cs-packet.i4px.com`). The workflow includes browser health checks, SSO login, data export, polling for completion, file download, and optional DingTalk delivery.

## Prerequisites

Create a configuration file `webhook_config.json` (managed by `xms_config_manager.py`):

```json
{
  "xms": {
    "url": "http://cs.packet.i4px.com/",
    "username": "YOUR_USERNAME",
    "password": "YOUR_PASSWORD"
  },
  "webhooks": [
    {
      "name": "Group Name",
      "token": "dingtalk_robot_token",
      "keyword": "required_keyword",
      "folderId": "https://alidocs.dingtalk.com/i/nodes/YOUR_FOLDER_ID",
      "enabled": true,
      "schedule": {
        "timeSlots": [{"hour": 9, "minute": 0}],
        "days": [1, 2, 3, 4, 5, 6, 0]
      }
    }
  ],
  "export": {
    "poll_interval_seconds": 120,
    "max_wait_minutes": 70,
    "file_name_template": "xms异常件导出_{timestamp}.xlsx",
    "max_retries": 3,
    "retry_interval_seconds": 60,
    "browser_max_recovery_attempts": 4,
    "send_notification_on_failure": true
  }
}
```

- `days`: 0=Sunday, 1=Monday, ..., 6=Saturday
- `poll_interval_seconds`: Seconds between progress checks
- `max_wait_minutes`: Max total wait time for export completion
- `max_retries`: Max full-workflow retry attempts when any phase fails
- `retry_interval_seconds`: Seconds to wait between retry attempts
- `browser_max_recovery_attempts`: Max browser recovery strategies to try per attempt
- `send_notification_on_failure`: Whether to send a DingTalk alert when all retries exhausted

## Retry & Error Handling

The automation runs inside an outer retry loop:

1. **Browser recovery first**: Each attempt begins with a 4-step browser health check (check size → JS resize → new tab → recreate window). If the browser MCP returns "Extension not connected" or `0x0`, the agent progressively escalates recovery strategies.
2. **Phase-level retries**: If login, export trigger, polling, download, or DingTalk upload fails, the agent records the error, waits `retry_interval_seconds`, and restarts the entire workflow from browser recovery.
3. **Failure notification**: If all `max_retries` are exhausted and `send_notification_on_failure` is true, the agent sends a DingTalk message to every matched webhook reporting the failure reason, so you are never left in the dark.

## Workflow

### Phase 0: Read Config & Filter Webhooks

1. Read `webhook_config.json`
2. Get current Beijing time (`Asia/Shanghai`) and day of week
3. For each webhook where `enabled=true`, check if current time matches any `timeSlots` and current day is in `days`
4. Collect matching webhooks. If none match and this is a scheduled run, exit silently

### Phase 0.5: Browser Window Health Check

Before any interaction, verify the browser window:

1. **Check size**: `window.innerWidth + ',' + window.innerHeight`
2. **JS resize**: If broken, `window.moveTo(0,0); window.resizeTo(1440,900)` then wait 2s and recheck
3. **New tab**: Call `tabs_create_mcp`, check new tab size
4. **Recreate window**: Close all tabs one by one (`tabs_close_mcp` only accepts a single `tabId` per call), then `tabs_context_mcp` with `createIfEmpty:true`
5. **Small window fallback**: If persistently small (e.g., 256x116) but not `0,0`, DOM operations still work. Only terminate if literally `0,0` after all steps

### Phase 1: Login

1. Navigate to `http://cs.packet.i4px.com/`
2. If redirected to `sso.i4px.com`, inject credentials via JavaScript:
   ```javascript
   document.getElementById('username').value = USERNAME;
   document.getElementById('passwordOrg').value = PASSWORD;
   ['input', 'change', 'keyup'].forEach(evt => {
     document.getElementById('username').dispatchEvent(new Event(evt, {bubbles:true}));
     document.getElementById('passwordOrg').dispatchEvent(new Event(evt, {bubbles:true}));
   });
   document.getElementById('signbtn').click();
   ```
3. **Handle "already logged in elsewhere" dialog**: If a dialog appears saying "此账号已在别的地方登录" or similar, click the "是"/"确定" button to continue
4. **Handle QR code / device verification**: If after clicking login, a QR code scan, mobile device authentication popup, or slider captcha appears that the Agent cannot complete automatically:
   - Do NOT retry repeatedly
   - Immediately notify the user via IM (e.g., 小Q channel)
   - Wait 3 minutes, then recheck if the page has redirected to XMS homepage
   - If still not redirected after timeout, record error and exit to failure notification
5. **Handle "server exception" error**: If at any point the page displays "服务器异常" (or "系统异常"/"服务异常"):
   - **Close the current tab** using `tabs_close_mcp`
   - **Create a new tab** using `tabs_create_mcp`
   - **Re-navigate** to `http://cs.packet.i4px.com/` in the new tab
   - **Restart the login flow** from Phase 1 Step 1
   - This counts toward the outer retry loop's `max_retries`
   > Do NOT repeatedly interact with the error page. Always close and recreate the tab.
6. **Login timeout**: If the login page URL does not change within 3 minutes and no special scenario above occurs, record "XMS login timeout" and handle via retry logic
7. Wait for redirect to `cs-packet.i4px.com/index`

### Phase 2: Export Data

1. Click **异常件管理** in left sidebar to expand submenu
2. Click **自有业务异常件管理**
3. **Clear the shelf organization field** (critical):
   - The visible field is `id=txtUpShelfOgCodeShow` (default value: "客户服务部")
   - The hidden field is `id=txtUpShelfOgCode` (default value: "D000414")
   - Clear both via JavaScript: `.value = ''` then dispatch `input`/`change` events
   - Alternative: left-click inside the input field on the right side of the text; the text should disappear automatically
4. Click **查询** (`id=abnormalSearch`)
5. Wait for data, then click **导出原因(新)** button (top-right)

### Phase 3: Poll for Completion

1. Click **下载中心** to open the export records dialog
2. Read the drawer table. First data row shows: task name, file size, progress %, status, export time, download action
3. If progress is not 100%, wait `poll_interval_seconds`, click **刷新**, and recheck
4. Repeat until 100% or `max_wait_minutes` exceeded

> **Modal reading tip**: `read_page` and `get_page_text` cannot see modal overlays. Use `computer` screenshot or JavaScript DOM queries on `.next-drawer` to read the export record table.

### Phase 4: Download

1. Read the **导出时间** from the first row (format: `2026-05-09 22:59:29`)
2. Generate filename: `xms异常件导出_2026-05-09 22_59_29.xlsx` (replace colons with underscores for Windows compatibility)
3. Click the **下载** button in the first row's action cell
4. Find the downloaded file in `~/Downloads/` (filename starts with `xms-cts-`)
5. Copy to output directory with the generated filename

### Phase 5: DingTalk Delivery (Optional)

For each matched webhook:

1. Call `get_file_upload_info` for upload credentials
2. Upload the xlsx via HTTP PUT to `resourceUrl`
3. Call `commit_uploaded_file` with `folderId` and filename to get `nodeId`
4. Generate permanent link: `https://alidocs.dingtalk.com/i/nodes/{nodeId}`
5. Send message via `send_message_by_custom_robot` including the webhook's keyword and the permanent link

## Verified Element References

| Element | ID / Text | Notes |
|---------|-----------|-------|
| SSO username | `id=username` | Not `loginName` |
| SSO password | `id=passwordOrg` | Not `loginPwd` |
| SSO login button | `id=signbtn` | Not `loginBtn` |
| Shelf org visible | `id=txtUpShelfOgCodeShow` | Default: "客户服务部" |
| Shelf org hidden | `id=txtUpShelfOgCode` | Default: "D000414" |
| Query button | `id=abnormalSearch` | |
| Export reason button | text=`导出原因(新)` | |
| Download center | text=`下载中心` | Opens drawer |
| Refresh button | text=`刷新` | In drawer |
| Drawer table | `.next-drawer table tr` | First data row = latest export |

## Critical Rules

- **Model**: When creating the QoderWork cron job, set `payload.model` to `"qwork-auto"` (standard model). Do not use flagship model for production.
- **Small window**: 256x116 is functional. DOM queries, `.click()`, and `.value` assignments work even in tiny windows. Only abort at `0,0`.
- **tabs_close_mcp**: Only accepts a single integer `tabId` per call. Close tabs one by one.
- **Filename**: Use the export record's "导出时间", not system time. Replace `:` with `_` for Windows filenames.
- **DingTalk links**: Always use `https://alidocs.dingtalk.com/i/nodes/{nodeId}`, never temporary OSS links.
- **Keywords**: Every robot message must contain that webhook's configured keyword.
- **Robot token parameter**: Use `robotToken` (not `access_token`) when calling `send_message_by_custom_robot`.
- **Upload Content-Type**: When uploading xlsx via HTTP PUT to `resourceUrl`, set `Content-Type` header to an empty string `""`, and include the returned `Authorization` and `x-oss-date` headers.
