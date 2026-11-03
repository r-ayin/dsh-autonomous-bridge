# dsh-autonomous-bridge

把 [`autonomous_studio`](https://github.com/) 的 planning 产物接成
[`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype)
的本地自动推进项。

> **works with** `deepseek-ai/dsh-superagent-prototype` @ `6337fc8`（MIT）
> 这是一个**外挂**扩展，不改上游任何代码。

---

## 它解决什么

`autonomous_studio` 有状态机（`.planning/status.json`）和阶段产物（requirements / prd /
test-cases / tech-plan），但它的引擎侧没有心跳实现；super agent 有执行能力，但没有
待办来源。两边都是空壳。

这个 bridge 补中间那层：

```
<repo>/.planning/status.json          状态机
        │  dispatch.py 读（过 7 类门禁）
        ▼
<repo>/.planning/pending/<proj>__<stage>__<ts>.task.md    任务书
        │  worker 执行（super agent automation 或人）
        ▼
<repo>/.planning/pending/<proj>__<stage>__<ts>.result.md  首行 STATUS: ok|partial|failed
        │  collect.py 读
        ▼
status.json 的 completedStages / currentStage / stageArtifacts / blockedReasons
```

## 为什么是外挂

上游是编译型 TS 项目，而且生产环境跑的就是它本人。bridge 只依赖文件系统与
`.planning/` 契约，做成独立 Python 脚本（纯标准库），对上游零影响，也不受构建链约束。
你随时可以删掉它，上游一行代码都不受影响。

## 快速开始

```bash
git clone <this repo> && cd dsh-autonomous-bridge

cp registry.json registry.mine.json
# 编辑：把 repo 指向含 .planning/status.json 的项目
#   { "id": "myproj", "repo": "/abs/path/to/project", "enabled": true, "maxPerDay": 6 }

bash selftest.sh                        # 先验证：应输出 ALL PASS (16 checks, 0 failed)
bash run_once.sh registry.mine.json     # 手动 tick 一次
python3 status_report.py registry.mine.json   # 看板
```

挂 cron（每 15 分钟）：

```cron
*/15 * * * * /abs/path/dsh-autonomous-bridge/run_once.sh /abs/path/registry.mine.json
```

worker 侧：把 `automation_prompt.md` 里的模板粘进一个 interval automation
（permission = `workspace-write`，每 15 分钟）。**只需要一个 automation**——
任务书数量增长不会带来 automation 数量增长。

## 文件

| 文件 | 作用 |
|---|---|
| `dispatch.py` | 心跳：读状态 → 过门禁 → 渲染任务书 → 幂等投递 |
| `collect.py` | 回收：读 result → 回写状态机 → 归档 |
| `run_once.sh` | 一次 tick：collect → dispatch → status_report |
| `status_report.py` | 多项目看板：阶段 / 卡点 / 待办数 / 末次 verdict |
| `automation_prompt.md` | 可直接粘进 automation 的 prompt 模板 |
| `selftest.sh` | 端到端自测，10 组用例 16 个断言 |
| `registry.json` | 受管项目列表 |

## 门禁（继承 autonomous-engine GATES，不放松）

| 条件 | 行为 |
|---|---|
| `currentStage == done` | 不再投递 |
| `engine.autoAdvance == false` | 全停（用户一句话即可关闭） |
| 阶段不在 `requirements/prd/tech-plan` | 转人工（除非项目显式 `allowAutoDevelopment`） |
| `draftPending.confirmed == false` | 等人确认，不投递 |
| `routeHealth.score < 5` 或 `correctionPending` | 路线修正中，不阻断用户但阻断自动推进 |
| 同 project+stage 已有未消费任务书 | 不重复投（幂等） |
| 当日投递 ≥ `maxPerDay` | 熔断转人工 |

硬红线，不因本工具而放松：**不自动 `git push`、部署阶段永不自动**。

## 自测覆盖

```
1  首次 dispatch 投递任务书（4 断言）
2  重复 dispatch 被幂等挡住（2）
3  result=ok 推进状态机 + 归档（2）
4  autoAdvance=false 全停
5  draftPending 未确认等待
6  routeHealth<5 阻断
7  development 阶段默认转人工
8  maxPerDay 熔断
9  result=partial 不推进、只记 blockedReasons
10 多项目隔离（缺失仓库跳过、看板可跑）（2）

ALL PASS (16 checks, 0 failed)
```

## 尚未覆盖

- 真实多项目并发下的调度压力（当前按顺序遍历 registry）
- result 的语义校验（只看首行 STATUS，不看产出质量）
- automation 侧的端到端联调（模板已给，但没做自动化测试）

## 与上游的关系

上游 `deepseek-ai/dsh-superagent-prototype` 目前关闭了 Issues（`has_issues: false`），
PR 端点不可用，所以本扩展暂以独立仓库形式存在。上游一旦开放，会提交为
`scripts/studio-bridge/` 目录。

MIT。
