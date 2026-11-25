# wechat-desktop-automation

微信桌面端（macOS 微信 4.x）自动化操作 WorkBuddy Skill。

通过 [WorkBuddy](https://www.workbuddy.cn) 内置的 computer-use 工具（视觉 advisor + 后台事件注入）完成微信桌面端的自动化收发操作，无需微信开放接口、不读取聊天数据库。

## 能做什么

| 场景 | 说明 |
|---|---|
| 会议纪要 / 日报 / 周报发群 | 整理定稿 → 预览确认 → 自动发送 |
| 文件 / 图片 / 报告转发 | 把本地文件按指令发给指定会话 |
| 定时提醒 | 配合定时任务，把提醒以"人话"发到文件传输助手或小群 |
| 群消息摘要 | 只读当前屏幕可见消息，结构化总结（不爬记录、不碰数据库） |
| 多群同步通知 | 同一通知逐群发送、逐次确认 |

## 核心链路（已实测跑通）

```
权限检查 → open -a WeChat 拉起窗口 → get_app_state → get_next_action（视觉决策）
→ 点击定位会话 → 点击输入框 → type_text（剪贴板粘贴，中文无损）→ 回车发送 → 截图复核确认
```

## 内置边界

- 只操作用户指定的会话，不点击其他会话 / 联系人 / 设置
- 发送内容零改动，不增删润色
- 只读取屏幕上看得见的内容做摘要，不爬聊天记录
- 微信出现登录 / 扫码 / 账号异常提示 → 立即停止并报告
- 支付、账号绑定等高危操作一律拒绝
- 多群发送逐群确认，不做无人监督批量

## 故障速查

| 现象 | 处理 |
|---|---|
| 窗口不出现 / 自动隐藏 | `open -a WeChat` 或 osascript activate 重新拉起 |
| `background mode requires valid 'pid' field` | 微信掉出注册表，重跑 list_apps 恢复 |
| 后台键鼠投递"未确认生效"警告 | 不重发，下一步截图让 advisor 复核实际状态 |
| 连续 2 次同一动作失败 | 停止重试，如实报告 |

## 安装（WorkBuddy 用户）

将 `SKILL.md` 和 `references/` 复制到 `~/.workbuddy/skills/wechat-desktop-automation/` 即可，重启会话后生效。

## 目录结构

```
wechat-desktop-automation/
├── SKILL.md                  # 主文件：核心循环 + 快速路径 + 故障速查 + 安全边界
└── references/
    └── scenarios.md          # 五类场景 SOP（确认级别 / 停止条件）
```

## 验证环境

- macOS + 微信 4.x（com.tencent.xinWeChat）中文界面
- WorkBuddy computer-use 插件（辅助功能 + 屏幕录制权限）
