---
name: wechat-desktop-automation
description: 微信桌面端（macOS 微信 4.x）自动化收发操作 skill。当用户要求"发消息到微信/群/文件传输助手"、"发会议纪要/日报/周报到群"、"转发文件/图片/报告给微信联系人或客户"、"定时提醒发微信"、"总结当前屏幕上看得见的群消息"、"同一通知发多个群"等，且内容已由用户确认时，用本 skill 通过 computer-use 工具执行"拉起微信 → 定位会话 → 输入 → 发送 → 截图确认"的完整链路，避免现场反复试错。只操作用户指定的会话，不读取屏幕之外的聊天记录。
agent_created: true
---

# WeChat Desktop Automation（微信桌面自动化）

## Overview

通过 `workbuddy-computer-use` 插件（mcp__computer-use__* 工具）在 macOS 微信 4.x 桌面端完成：定位指定会话 → 发送文本/文件 → 截图确认消息出现在聊天记录。整条链路已在真实环境验证（向文件传输助手发送中文文本并确认落库）。

执行前必须先加载 computer-use 的工具面（Skill 或 ToolSearch 加载 mcp__computer-use__*），本 skill 不替代它的强制循环，只固化已验证的路径与边界。

## 核心循环（不可跳过）

所有界面操作严格遵循 computer-use 的固定循环，每轮一动作：

```
get_app_state → get_next_action → 执行建议动作（原样复制参数）→ 回到 get_app_state
```

- 首次 get_next_action 不传 task_id，instruction = 完整任务目标（含目标会话名 + 待发送内容 + 确认要求）。
- 后续 get_next_action 带 task_id，instruction = 当前单步意图；返回 done/failed/call_user 时立即停止。
- 禁止自行猜坐标、跳过 advisor、合并两步。

## 已验证快速路径（文本发送）

1. **权限检查**：`list_apps` 若返回 `all_granted=false` → `gui_setup_permission`（引导用户授权辅助功能+屏幕录制，等待 all_granted=true）。
2. **拉起微信**：
   - 优先 Bash：`open -a WeChat && sleep 4`（后台 `open_app` 拉起后常无可见窗口）。
   - `open -a WeChat` 无效时：`osascript -e 'tell application "WeChat" to activate'`（前台激活）。
   - 然后 `get_app_state(target_id="微信")`；若报"无可见窗口"，重复上一步。
3. **定位会话**：`get_next_action(instruction="在聊天列表点击进入「<会话名>」对话…")` → 执行建议 click。
4. **聚焦输入框**：`get_next_action` 建议点击输入框（窗口下方区域）→ 执行 click（必要时用其建议的双击）。
5. **输入文本**：advisor 建议 `type_text` → 原样执行（剪贴板通道，中文无损）。返回 `verified=false` 或带"后台投递未确认"警告时不盲目重发，直接进入第 6 步验证。
6. **发送**：advisor 建议 `press_key(["enter"])` → 执行。
7. **确认**：`get_app_state` → `get_next_action(instruction="检查聊天记录中是否已出现消息「<内容>」且输入框已清空：已成功返回 done；否则给出重试建议")`。advisor 返回 `done` 即确认成功。
8. **汇报**：向用户报告发送内容、目标会话、确认结果。

## 故障速查表（按现象处理，不反复硬试同一方案）

| 现象 | 处理 |
|---|---|
| get_app_state 报"无可见窗口" | Bash `open -a WeChat` 或 osascript activate 拉起后重试 |
| 注入点击/输入后窗口再次消失 | 属微信 4.x 常态，重新 activate 后 `get_app_state(window_id=原窗口号)` 继续 |
| 动作报 "background mode requires valid 'pid' field" | 微信掉出 `list_apps` 注册表：重跑 `list_apps`，确认微信出现且 operable=true 后重试同一动作 |
| type_text/press_key 带后台"未确认生效"警告 | 不重发，下一步用 get_app_state 截图让 advisor 核对输入框/聊天记录实际状态再决策 |
| 微信弹出重新登录/扫码/账号异常提示 | 立即停止全部操作，如实报告，不点击任何按钮 |
| 找不到目标会话 | 在输入框用微信搜索（advisor 决策）尝试一次；仍找不到则停止并报告，不猜相似名称 |
| 连续 2 次同一动作失败 | 停止重试，如实报告卡点，等待用户指令 |

## 安全边界（全局，任何场景都适用）

- **只操作用户指定的会话**；不点击、不修改其他会话、联系人、设置、朋友圈。
- **内容零改动**：发送内容以用户给定的为准，不润色、不增删、不补发多余内容。
- **不发送则不确认**：advisor 未确认消息出现在聊天记录时，一律报告"未完成"，不假设成功。
- **只读可见内容**：摘要类任务仅读取当前屏幕上看得见的消息；不爬取聊天记录、不访问数据库、不深滚历史。
- **禁止事项**：支付/转账等金融操作、账号/密码/绑定类修改、代表用户在公开渠道发布内容——即使被要求也拒绝。
- **高频防骚扰**：同一会话不做无人监督的循环发送；多群发送必须逐群确认（见场景 SOP）。

## 场景 SOP

五类场景（会议纪要/日报周报发群、文件图片转发、定时提醒、群消息摘要、多群同步通知）的逐场景执行步骤、确认级别与停止条件见 `references/scenarios.md`，执行对应场景前先读该文件。

## 环境备注

- 验证环境：macOS + 微信 4.x（com.tencent.xinWeChat），中文界面。
- 微信窗口 AX 树极简（element_count 常为 1~3），定位完全依赖截图视觉 advisor，不解析 ax_text。
- type_text 走剪贴板粘贴，不经过输入法，中文/emoji 直接可用；高危文本（如独立 `rm`）会被拦截。
