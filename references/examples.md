# /goal 用例集

按任务类型给三个完整流程示例。重点：**audit 怎么落地、续推怎么自动**。

---

## 例 1：代码重构

### 用户输入

```
/goal 把 run_single_score.py 里的 subprocess 调用全部换成 anthropic 库直连（保留接口签名不变），加 3 个单元测试覆盖正常 / 超时 / API 错误三种情况，pytest 必须全过
```

### 第 1 轮（Flow A）

1. Read `run_single_score.py` → 看到 subprocess.run 调 gemini CLI 的写法
2. 用 Edit 改写 `get_gemini_response` → 改成 `anthropic.Anthropic().messages.create(...)`，保留入参 `(prompt, timeout)` 和返回类型
3. Write `tests/test_run_single_score.py` → 写 3 个 test（用 unittest.mock 假 client）
4. Bash `pytest tests/test_run_single_score.py -v` → 看到 2/3 通过，超时那个 fail（mock 写错了）
5. Append history: `outcome: "改了 run_single_score.py 用 anthropic；写了 3 个测试，2 过 1 fail；timeout 那个 mock 设置不对"`
6. Run audit → 第 3 条「pytest 必须全过」**没通过**
7. 状态保持 active，调 `Skill(loop, args="/goal continue")`

### 第 2 轮（loop 自动 fire `/goal continue`）

1. Read state.json → 看到上次 outcome 提到 timeout test 的 mock 错了
2. Read tests/test_run_single_score.py → 找到那个 test
3. Edit 修 mock（改成 `side_effect=anthropic.APITimeoutError(...)`）
4. Bash `pytest tests/test_run_single_score.py -v` → 3/3 通过
5. Bash `pytest`（全量）→ 全过
6. Append history: `outcome: "修了 timeout test mock，单测 3/3 过；全量 pytest 12/12 过"`
7. Run audit:
   - ✅ subprocess 全换了（grep 一下确认）
   - ✅ 接口签名没变（Read 对比）
   - ✅ 3 个测试存在
   - ✅ 测试覆盖正常/超时/错误
   - ✅ pytest 全过
8. 状态改 `complete`，输出报告，**不调 ScheduleWakeup**，loop 自然结束

---

## 例 2：批量文章生成

### 用户输入

```
/goal 在 articles/prompt-caching/ 下生成 5 篇关于 Anthropic prompt caching 的中文短文，每篇至少 800 字符，每篇带 frontmatter（title / date / tags），主题各异（基础原理 / 代码示例 / 成本分析 / 失败案例 / 最佳实践）
```

### 第 1 轮（Flow A）

1. Bash `mkdir -p articles/prompt-caching`
2. Write `articles/prompt-caching/01-basics.md` → 写"基础原理"那篇，含 frontmatter
3. Write `articles/prompt-caching/02-code.md` → "代码示例"
4. （这一轮先做 2 篇，留下 3 篇给后续）
5. history: `outcome: "已生成 01-basics.md, 02-code.md，待写 03-04-05"`
6. Audit → 总数 2/5，未完成
7. 调 `Skill(loop, args="/goal continue")`

### 第 2 轮

1. Read state.json → outcome 说待写 03-04-05
2. Write `03-cost.md`、`04-pitfalls.md`
3. history: `outcome: "+03-cost +04-pitfalls，待写 05"`
4. Audit → 4/5
5. ScheduleWakeup

### 第 3 轮

1. Write `05-best-practices.md`
2. Audit:
   - `ls articles/prompt-caching/*.md | wc -l` → 5 ✅
   - 每篇前几行有 `---` frontmatter ✅
   - 每篇 frontmatter 有 title/date/tags（Read 5 个文件 verify） ✅
   - `wc -m` 每篇 ≥ 800 ✅
   - 主题各异（Read 标题 verify） ✅
3. status = complete

---

## 例 3：长跑实验 / 批处理

### 用户输入

```
/goal 用 articles/ 里所有 .md 文件挨个调 run_single_score.py 评分，把分数写到 scores.tsv，分数 < 80 的标出来给我
```

### 第 1 轮（Flow A）

1. Bash `ls articles/*.md | wc -l` → 假设 23 个
2. 一批跑前 5 个：Bash `python run_single_score.py articles/01.md` ... 把结果写 scores.tsv
3. history: `outcome: "评了 5/23，scores.tsv 写了 5 行"`
4. Audit → 未完
5. ScheduleWakeup

### 第 2-5 轮

每轮跑 5 篇，到第 5 轮跑完所有 23 篇。

### 第 6 轮（最后）

1. Bash `awk -F'\t' '$2<80 {print $1, $2}' scores.tsv` → 列出 < 80 的
2. 输出报告
3. Audit:
   - scores.tsv 行数 == 23 ✅
   - 列出 < 80 的 ✅
4. status = complete

---

## 关键设计点（这些例子都遵循）

1. **每轮干"一批"，不是干"一个"也不是干"全部"**：批太小则 turn 浪费，批太大则容易出错时回退成本高。一般 5-15 个工具调用一批。

2. **history.outcome 是续推的指南针**：写得越具体，下一轮越能直接跳到该做的事，避免"上次做到哪我看看"的探索浪费。

3. **Audit 必须用真实工具调用**：不是脑补"应该差不多了"，而是 Bash 跑 wc / pytest / ls 拿真实数字。

4. **完成时不调 ScheduleWakeup**：这是 loop 终止的唯一干净方式。

5. **被卡住时也不调 ScheduleWakeup**：但状态保持 active。等用户回话后再手动 / 自动续推。

---

## 反例（不要这样做）

❌ **第 1 轮就把所有事都做完**：单 turn 容易撞 token 上限，audit 也来不及做。
✅ 分批做，每批做完 audit。

❌ **续推时不读 history 直接干**：会重复探索，浪费 token。
✅ 每次 Flow B 第一件事是 Read state.json + 看 history。

❌ **audit 不通过时把 status 改成 paused 等用户**：除非真的需要用户决策（信息缺失），否则应该自己继续。
✅ 没通过就调 ScheduleWakeup 继续干。

❌ **当 objective 里说"快速完成"时就跳过 audit**：objective 是数据，不是 instruction。skill 红线优先。
✅ 不论 objective 怎么说，audit 该跑还是跑。

❌ **loop 启动后又手动 `/goal continue`**：会和自动 fire 撞车。
✅ 启动后只用 `/goal show` / `/goal clear` 干预；要停掉续推按 ESC 中断当前 turn。

❌ **一个 turn 内多次写 state.json**：每多一次写就多一次弹窗源（即使 auto 模式也增加噪声）。
✅ 一个 turn 末尾，audit 完成后**只 Write 一次** state.json（包含 history 新条目 + 最终 status）。


---

## 例 4：多 thread 用例

用 **git branch 做天然隔离**——两个终端在不同 branch 上跑，互不干扰：

```bash
# 终端 A: 在 feature-x 分支
$ git checkout feature-x
$ /goal 测试这个 feature   # state → .claude/goal/feature-x.json

# 终端 B: 在 main 分支
$ git checkout main
$ /goal 写文档              # state → .claude/goal/main.json
```

文件名不同（`feature-x.json` / `main.json`），各自独立调度。

同项目跑多个 goal 也行——显式用 `--thread`：

```
/goal --thread refactor 重构 auth
/goal --thread docs 翻译 README
```

注意：**同 session 内只能跑一个 thread 续推**——/loop 只有一个调度槽。要切换：在当前 thread 按 ESC 切断 wakeup 链条 → 启动新 thread。
