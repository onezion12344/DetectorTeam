# Pitfalls — 一劳永逸固化记录

## 2026-07-22: chatlog `head` truncation cuts off newest messages

**症状**: 查询 filehelper 对话，显示的消息只到 7/18，用户 7/22 早上发的消息全部丢失
**根因**: API 返回 newest-first，reversed 后 `head -150` 把最新的 50 条（共 200）裁掉了。另外 chatlog serve 中途 DB 连接断开，第二次查询返回 0 条
**修复**: 
  - SKILL.md 加 HARD RULES：禁止 pipe 到 head/tail，用 limit 参数控制数量
  - 加 date range verification：每次查询后打印消息时间范围
  - 加 0-message 自愈模式：total_count=0 时自动重启 serve
  - 默认 newest-first 展示，需要时序时再 reverse
**验证**: 重启后直接查到 7/22 最新消息
**看哪里**: `~/.claude/skills/onezion-chatlog/SKILL.md`
