# claude-goal-skill

一个 Claude Code skill，把 OpenAI Codex CLI 的 [`/goal`](https://github.com/openai/codex/tree/main/codex-rs/core/templates/goals) 命令移植到 Claude Code 上。

> 你给一个长期目标，它一轮一轮自己往前推，直到完成或被暂停。

## 能干什么

```
/goal 把 articles/ 下所有 .md 加上 frontmatter
   ↓
状态写到 .claude/goal/main.json
   ↓
干第一批活 → 跑 audit → 没完成
   ↓
启动 /loop → 自动一轮一轮往前推
   ↓
audit 通过 → 标 complete → 自动停
```

中间用户可以随时干预：

| 命令 | 效果 |
|---|---|
| `/goal <任务>` | 启动新 goal |
| ESC | 中断当前 turn → 切断续推链条 |
| `cat .claude/goal/<id>.json` | 看进度 |
| `rm .claude/goal/<id>.json` | 清状态 |

## 装载

```bash
# 项目级（只这个项目能用）
git clone https://github.com/limin112/claude-goal-skill .claude/skills/goal

# 全局（所有项目都能用）
git clone https://github.com/limin112/claude-goal-skill ~/.claude/skills/goal
```

装完重启 Claude Code，输入 `/goal <你的目标>` 即可。

**强烈建议**：先按 `Shift+Tab` 开 Claude Code 的 Auto Accept Edits 模式——续推过程中不会被弹窗打断。

## 它跟 Codex 原版 `/goal` 的对应

| Codex 原版 | 本 skill |
|---|---|
| `continuation.md` 注入 prompt | `references/completion-audit.md`（核心 prompt 几乎照抄） |
| `update_goal(complete)` 工具（enum 锁） | SKILL.md 里的红线 + audit gate |
| `thread_goals` SQLite 表 | `.claude/goal/<thread_id>.json` 一文件一 thread |
| 4 状态机（含 budget_limited） | 3 状态（active/paused/complete），budget 交给 Claude Code 的 `/compact` |
| Codex runtime 自动续推 | `Skill(loop, ...)` + `ScheduleWakeup` |
| 中断 = 自动 pause | Flow B 的「话题变更检测」+ 用户 ESC |
| 多 thread 并发（thread_id 主键） | 不同 JSON 文件天然隔离；同 session 强制单活跃 thread |

## 设计哲学

1. **状态轻**：JSON 文件代替 SQLite，Read/Write 直接操作
2. **续推靠复用**：把 Claude Code 内置的 `/loop` 包进来，不自造调度器
3. **完成判定要狠**：抄 Codex 的 completion audit prompt——不让模型用「我做了好多努力」当完成证据
4. **多 thread 用文件名隔离**：每个 thread 一个 `.json`，不在单文件里 squeeze
5. **每 turn 状态文件只写一次**：减少 UX 噪声

## 文件结构

```
SKILL.md                       # 入口指令书，给 Claude 看
references/
├── completion-audit.md        # 完成判定 prompt（按需加载）
└── examples.md                # 用例集（按需加载）
```

## 致谢

- OpenAI [Codex](https://github.com/openai/codex) — 原版 `/goal` 的设计、prompt、状态机
- Anthropic [Claude Code](https://claude.com/claude-code) — skill 系统 + `/loop` + `ScheduleWakeup`

## 文章

详细的设计过程 / 踩坑 / 拆 Codex prompt：[抄 Codex 的 /goal 到 Claude Code（待发布）](#)

## License

MIT
