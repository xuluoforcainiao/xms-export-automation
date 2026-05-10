# XMS 异常件导出自动化 — 使用手册

## 简介

本工具用于自动化导出 XMS 客服管理系统（`cs-packet.i4px.com`）中的「自有业务异常件」数据，并将导出的 Excel 文件自动上传到钉钉群。支持多群独立配置执行时间、关键词和文件夹，支持每日定时自动执行，也支持手动触发单次导出。

## 功能特性

- **全自动 SSO 登录**：自动检测并跳转 SSO 登录页，通过 DOM 注入完成免交互登录
- **智能上架组织清空**：自动清空「上架组织」筛选条件，确保导出全量数据
- **异步导出等待**：提交导出任务后自动轮询「下载中心」，等待生成完成
- **多群独立推送**：每个钉钉群可独立配置执行时间、关键词和目标文件夹
- **可视化配置管理**：提供本地 Web GUI（`xms_config_manager.py`），零代码修改配置
- **浏览器窗口自愈**：内置四级窗口恢复策略，应对浏览器环境异常

## 系统要求

- Windows 10/11
- Python 3.8+
- QoderWork 桌面端（用于托管定时任务 Agent）
- XMS 系统账号（拥有「自有业务异常件管理」菜单权限）
- 钉钉自定义机器人（用于群消息推送）

## 安装步骤

### 1. 准备配置文件

使用 `xms_config_manager.py` 创建并管理 `webhook_config.json`，内容如下：

```json
{
  "xms": {
    "url": "http://cs.packet.i4px.com/",
    "username": "YOUR_XMS_USERNAME",
    "password": "YOUR_XMS_PASSWORD"
  },
  "webhooks": [
    {
      "name": "运营通知群",
      "token": "YOUR_DINGTALK_ROBOT_TOKEN",
      "keyword": "自有业务异常件",
      "folderId": "https://alidocs.dingtalk.com/i/nodes/XXXXX",
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

字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `xms.username` | string | XMS 登录账号 |
| `xms.password` | string | XMS 登录密码 |
| `webhooks[].name` | string | 群名称（仅用于展示） |
| `webhooks[].token` | string | 钉钉机器人 access_token |
| `webhooks[].keyword` | string | 机器人安全设置中的关键词 |
| `webhooks[].folderId` | string | 钉钉文档文件夹链接（用于文件归档） |
| `webhooks[].enabled` | boolean | 是否启用该群推送 |
| `webhooks[].schedule.timeSlots` | array | 执行时间点列表，可配置多个 |
| `webhooks[].schedule.days` | array | 执行星期（0=周日，1=周一...6=周六） |
| `export.poll_interval_seconds` | int | 导出后轮询间隔（秒） |
| `export.max_wait_minutes` | int | 最大等待导出完成时间（分钟） |
| `export.file_name_template` | string | 文件名模板，`{timestamp}` 会被替换为导出时间 |
| `export.max_retries` | int | 工作流失败后的最大重试次数（默认 3） |
| `export.retry_interval_seconds` | int | 每次重试前的等待间隔（默认 60 秒） |
| `export.browser_max_recovery_attempts` | int | 浏览器恢复的最大尝试策略数（默认 4） |
| `export.send_notification_on_failure` | boolean | 全部重试失败后是否发送钉钉失败通知（默认 true） |

### 2. 使用可视化配置工具（推荐）

双击运行 `xms_config_manager.py`，会自动打开浏览器并显示配置管理界面。

界面功能：
- **全局执行计划**：顶部展示所有启用 Webhook 的时间并集，以及对应的 Cron 表达式
- **Webhook 卡片**：每个群独立展示，可开关、删除、配置执行时间和执行日
- **添加 Webhook**：填写群名称、Token、关键词即可新增
- **保存配置**：点击「保存配置」按钮将修改写入文件
- **复制同步指令**：当修改了执行时间后，点击此按钮复制指令，粘贴到 QoderWork 对话框发送，完成 Cron 调度同步

### 3. 设置定时任务（Cron）

在 QoderWork 中发送同步指令，格式为：

```
同步cron: 0 9 * * *
```

其中 `0 9 * * *` 是 Cron 表达式，表示每天上午 9:00 执行。该表达式应与配置文件中「全局执行计划」显示的 Cron 一致。

Cron 表达式规则：
- `分 时 日 月 周`
- 示例：`0 9 * * *` = 每天 9:00
- 示例：`0 9,14 * * 1-5` = 工作日 9:00 和 14:00

### 4. 首次手动测试

配置完成后，建议在 QoderWork 中发送：

```
触发一次手动测试
```

Agent 会立即执行一次完整流程，包括登录、导出、下载和钉钉推送，用于验证配置是否正确。

## 工作流程详解

Agent 执行时按以下阶段运行：

### Phase 0 — 配置读取与过滤

读取 `webhook_config.json`，获取当前北京时间（Asia/Shanghai）和星期几。遍历所有 Webhook，检查 `enabled` 是否为 true、当前时间是否匹配 `timeSlots`、当前星期是否在 `days` 中。收集所有匹配的 Webhook，如果没有匹配项则静默退出。

### Phase 0.5 — 浏览器窗口健康检查

在操作页面前，先检查浏览器窗口状态：

1. **检查尺寸**：通过 JS 获取 `window.innerWidth` 和 `window.innerHeight`
2. **JS 恢复**：如果窗口异常（0x0 或极小），尝试 `window.resizeTo(1440, 900)`
3. **新建标签页**：如果仍异常，创建新标签页并检查
4. **重建窗口**：关闭所有旧标签页，调用 `tabs_context_mcp` 重建浏览器窗口
5. **小窗口兜底**：256x116 的极小窗口仍可正常进行 DOM 操作，仅在 0x0 时终止

### Phase 1 — SSO 登录

打开 XMS 地址，如果跳转至 `sso.i4px.com`，通过 JavaScript DOM 注入完成登录：

```javascript
document.getElementById('username').value = USERNAME;
document.getElementById('passwordOrg').value = PASSWORD;
['input', 'change', 'keyup'].forEach(evt => {
  document.getElementById('username').dispatchEvent(new Event(evt, {bubbles:true}));
  document.getElementById('passwordOrg').dispatchEvent(new Event(evt, {bubbles:true}));
});
document.getElementById('signbtn').click();
```

等待页面跳转回 `cs-packet.i4px.com/index` 确认登录成功。

> 注意：SSO 登录页的元素 ID 为 `username`、`passwordOrg`、`signbtn`，不是 `loginName`/`loginPwd`/`loginBtn`。

### Phase 2 — 导出数据

1. 点击左侧菜单「异常件管理」展开子菜单
2. 点击「自有业务异常件管理」
3. **清空上架组织**：找到「上架组织」输入框（默认显示「客户服务部」），点击输入框内部右侧使文字自动消失。若未消失，通过 JS 清空 `txtUpShelfOgCodeShow` 和 `txtUpShelfOgCode` 两个字段并触发 `input`/`change` 事件
4. 点击「查询」按钮（`id=abnormalSearch`）
5. 等待数据加载完成后，点击右上角「导出原因（新）」按钮

### Phase 3 — 轮询导出进度

1. 点击「下载中心」打开导出记录弹窗
2. 读取第一行数据的「进度」列
3. 若未达到 100%，等待 `poll_interval_seconds` 秒后点击「刷新」重新检查
4. 重复直到进度 100%，或超过 `max_wait_minutes` 分钟则记录超时

> 弹窗内的表格对 `read_page` 不可见，需通过 `computer` 截图或 JavaScript DOM 查询 `.next-drawer table tr` 读取。

### Phase 4 — 下载文件

1. 读取第一行「创建时间」列（格式如 `2026-05-09 22:59:29`）
2. 生成文件名：`xms异常件导出_2026-05-09 22_59_29.xlsx`（冒号替换为下划线）
3. 点击「下载」按钮
4. 在 `~/Downloads/` 中找到下载的文件（文件名以 `xms-cts-` 开头）
5. 重命名为步骤 2 确定的文件名

### 异常重试与失败通知

整个工作流包裹在一个外层重试循环中：

1. **浏览器自愈优先**：每次重试开始时，Agent 会先执行浏览器窗口健康检查（检查尺寸 → JS resize → 新建标签页 → 重建窗口）。如果浏览器 MCP 返回 `Extension not connected` 或窗口为 `0x0`，Agent 会逐级尝试恢复。
2. **阶段级重试**：如果登录、导出触发、轮询、下载或钉钉推送任一阶段失败，Agent 会记录错误原因，等待 `retry_interval_seconds` 后重新从浏览器恢复开始执行。
3. **失败兜底通知**：当所有 `max_retries` 次尝试均失败后，如果 `send_notification_on_failure` 为 true，Agent 会向所有匹配的钉钉群发送失败提醒，包含错误信息，避免「静默失败」。

### Phase 5 — 钉钉推送

对每个匹配的 Webhook：

1. 调用 `get_file_upload_info` 获取上传凭证
2. 通过 HTTP PUT 将文件上传至 `resourceUrl`
3. 调用 `commit_uploaded_file`，传入 `folderId` 和文件名，获取 `nodeId`
4. 生成永久链接：`https://alidocs.dingtalk.com/i/nodes/{nodeId}`
5. 调用 `send_message_by_custom_robot` 发送群消息，消息内容必须包含该 Webhook 的 `keyword`

## 常见问题排查

### 1. 配置修改后未生效

`xms_config_manager.py` 保存的配置会同时写入三个位置：
- 当前目录的 `webhook_config.json`
- 当前目录的 `webhook_config.json`（唯一正确的配置文件）

确保 QoderWork 定时任务的 `contextDirs` 指向了正确的配置目录。

### 2. 定时任务未触发

检查 QoderWork Cron 的 `schedule.expr` 是否与配置文件的「全局执行计划」一致。若修改了执行时间但未同步 Cron，Agent 不会在正确的时间被唤醒。

### 3. SSO 登录失败

确认账号密码正确，且未被 SSO 强制要求修改密码。若页面元素 ID 发生变化，可通过浏览器开发者工具检查实际 ID 并更新 SKILL.md 中的元素参考表。

### 4. 上架组织未清空

不要点击左侧「上架组织」标签文字，必须点击输入框内部。如果文字未自动消失，Agent 会自动通过 JS 强制清空并触发事件。

### 5. 弹窗内容读取不到

`read_page` 和 `get_page_text` 无法读取 Modal/Drawer 弹窗内容。Agent 使用 `computer` 截图结合 JS DOM 查询 `.next-drawer` 来读取进度和创建时间。

### 6. 钉钉推送失败

- 检查机器人 Token 是否正确
- 检查消息内容是否包含机器人安全设置中的关键词
- 检查 `folderId` 是否为有效的钉钉文档文件夹链接
- 检查机器人是否被禁言或从群中移除

## 文件结构

```
C:/Users/浩正/Desktop/AI工具/库内异常件/
├── webhook_config.json         # 唯一正确的配置文件（Agent 运行时读取）
├── xms_config_manager.py       # 可视化配置管理工具
└── xms异常件导出_*.xlsx        # 导出的数据文件（按时间命名）

C:/Users/浩正/.qoderwork/skills/xms-export-automation/
├── SKILL.md                    # Skill 主文档（工作流程和元素参考）
├── config-template.json        # 配置模板
├── README.md                   # 使用手册（本文档）
└── CHANGELOG.md                # 开发日志

C:/Users/浩正/.qoderwork/skills/xms-login/
└── SKILL.md                    # SSO 登录 Skill（独立模块）
```

## 注意事项

- **模型选择**：Cron 任务的 `payload.model` 必须设为 `qwork-auto`（标准模型），不要使用旗舰模型，否则可能因工具调用策略差异导致流程失败。
- **敏感信息**：配置文件包含明文密码和钉钉 Token，请勿将配置文件上传至公共仓库。
- **浏览器复用**：每次 Agent 运行结束后会保留浏览器标签页，下次运行时会复用已有窗口。如果窗口损坏，Agent 会自动执行恢复策略。
- **文件命名**：下载后的文件名使用导出记录中的「创建时间」，而不是系统当前时间，确保文件名与实际数据批次一致。
- **钉钉链接**：发送给群友的文件链接必须使用 `commit_uploaded_file` 返回的永久节点链接，不要使用 PUT 上传后的临时 OSS 链接（15 分钟过期）。
