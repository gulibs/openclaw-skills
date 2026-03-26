---
name: nba-parlay-steps
description: >-
  Pull nba_analysis git repo and run OpenClaw step files 00-06 for Beijing game-day parlay analysis;
  write structured JSON under output/. Use when user provides run_id, data_bundle_date, beijing_game_date.
compatibility: Requires git on the Gateway host PATH and a git clone at NBA_ANALYSIS_ROOT (or ~/nba_analysis).
metadata:
  openclaw:
    requires:
      bins:
        - git
---

# NBA 串关分步分析（nba_analysis 数据包）

## 何时使用

用户给出 **`run_id`**、**`data_bundle_date`**（数据包目录日 `YYYY-MM-DD`）、**`beijing_game_date`**（要分析的比赛北京日历日），且数据已落在 git 仓库 **`NBA_ANALYSIS_ROOT`**（未设置时默认 `~/nba_analysis`）路径：

`{NBA_ANALYSIS_ROOT}/{data_bundle_date}/runs/{run_id}/`

## 必须执行的前置

1. **`NBA_ANALYSIS_ROOT` 必须是 git 工作区**（根目录存在 `.git`）。若 `git pull` 报 `fatal: not a git repository`，需先在该路径 `git clone <远端>` 或 `git init` 并配置 `remote`，不可使用普通空文件夹。
2. **每次开始任务前**在仓库根目录执行：`cd {NBA_ANALYSIS_ROOT} && git pull`（或 `git pull --ff-only`），确保与远端一致。分析服务侧可能对同一 **run_id** 做 **重新落盘（覆盖）** 或 **删除目录并 push**；若本地不 pull，仍停留在旧目录或已删路径，会导致读错文件或路径不存在。
3. 打开上述目录下的 **`manifest.json`**，读取 `steps` 数组。

**与后端的同步**：分析服务在本机用 **`NBA_ANALYSIS_GIT_ROOT`** 写入并 push 到远端；你在 **Gateway 主机** 上的 clone 拉的是同一远端。**Hooks / 管理端下发给模型的任务正文只含 `run_id` 与相对路径** `{data_bundle_date}/runs/{run_id}/`，**不会**带分析服务器上的绝对路径（如 macOS `/Users/...`）；请始终用本机的 **`NBA_ANALYSIS_ROOT`**（或 `~/nba_analysis`）作为仓库根，再拼上上述相对路径。远端更新（含你写入的 `output/`、或服务端 reprepare/删除的 commit）后须 **`git pull`** 再执行步骤。

## 执行顺序（严格）

按 `manifest.steps` 中 **`order` 升序** 对应文件执行（与文件名 `00_`、`01_`…`06_` 一致）：

1. **00** `00_README.md` — 理解分步说明。
2. **01** `01_rules.md` — 引擎规则。
3. **02** `02_schedule.md` — 赛程完整性；Fail Fast 则停止。
4. **03** `03_odds_index.md` — 盘口分级索引。
5. **04** — **每个** `04_game_*.md` **单独一轮**（一场一轮），不得跳过顺序。
6. **05** `05_synthesis_parlay.md` — 串关合成。
7. **06** `06_output_template.md` — 最终排版（可与 05 合并）。

**分析对象语义**：所有结论针对 **`beijing_game_date`（北京时间比赛日）**；`manifest.target_date` 仅为数据包生成时的赛程日，若与用户指定的 `beijing_game_date` 不同，**以用户指定的比赛日为准**重新框定「待分析场次」（在数据包场次范围内筛选该日的比赛，或明确说明数据包不包含该日则终止）。

## 数据边界与措辞（与后端 `data_compact` 一致）

- 数据包**不包含**内部 `model_predictions` / `edge_results`（ensemble 胜率、EV 等）。输出中**禁止**用「**模型预测**」指代上述内容。
- **`expected_total_points`**（若出现在 `analysis_signals` 或特征中）是**两队近 10 场场均得分之和**的启发式，**不是**单独一条模型推理管线；应写「**预期总分（滚动场均之和）**」等，勿写「模型预测 xxx 分」。
- **排名**、`rest_days`、**体彩赔率**见各步 JSON；`features_subset` 已剔除与 `odds_lines` 重复的 `market` / `line_movement`，避免与「隐含概率」表述混淆。

## 输出契约（必须写入磁盘）

在运行目录下创建子目录 **`output/`**（与 `manifest.json` 同级）：

### `output/meta.json`

```json
{
  "schema_version": 1,
  "status": "completed",
  "run_id": "<uuid>",
  "data_bundle_date": "YYYY-MM-DD",
  "beijing_game_date": "YYYY-MM-DD",
  "finished_steps": ["00_README", "01_rules", "02_schedule", "03_odds_index", "04_game_*", "05_synthesis", "06_output"],
  "updated_at_utc": "<ISO8601 Z>"
}
```

`status`：`in_progress` | `completed` | `failed`。

### `output/parlay_recommendation.json`

```json
{
  "schema_version": 1,
  "run_id": "<uuid>",
  "data_bundle_date": "YYYY-MM-DD",
  "beijing_game_date": "YYYY-MM-DD",
  "summary_zh": "一两句总述",
  "report_markdown_zh": "## 【比赛清单】\\n…（可选，见下）",
  "parlays": [
    {
      "label": "保守|中风险|激进",
      "legs": [
        {
          "game_id": 0,
          "matchup_zh": "湖人（主） vs 老鹰（客）",
          "home_team": "湖人",
          "away_team": "老鹰",
          "pick_zh": "让分示例：（客）老鹰 vs 湖人（-10.5）· 让分主胜（主队赢盘）；勿写「湖人让分(-10.5)」不写对阵",
          "bet_type_hint": "rfsf",
          "rationale_zh": "理由",
          "confidence_1_to_10": 7,
          "risk_notes_zh": "风险"
        }
      ],
      "combined_rationale_zh": "组合逻辑",
      "max_risk_zh": "最大风险说明"
    }
  ],
  "disclaimer_zh": "非投注建议；数据来自爬虫，以体彩官网为准。"
}
```

- **`report_markdown_zh`（可选但强烈推荐）**：字符串，完整 Markdown 报告，建议包含与步骤 6 骨架一致的章节，例如「【比赛清单】」「【单场分析摘要】」「【串关组合】」「【终止条件检查】」「【最终投注建议】」及元信息（分析完成时间、数据源说明）。管理端 `/admin/openclaw-parlay-prompt` 在同步入库后会**渲染该字段**为解析报告，并与下方结构化 `parlays` 卡片并列展示。若省略，页面仅显示结构化串关卡片。JSON 内换行使用 `\n` 或真实换行均可。
- `parlays` 可为空数组（无合格串关时）。
- 每条 leg **必须**含 **`matchup_zh`**（推荐）或并列 **`home_team`** / **`away_team`**，与步骤 2 赛程表一致；**禁止**只写玩法与盘口线而不写对阵。后端同步入库时会按 `game_id` 从库表补全队名，并尽量补 **`spread_line` / `rfsf_board_zh`（（客）客 vs 主（±线））**，但 **`pick_zh` 仍须**按体彩习惯自写让分句。
- **让分 `pick_zh`**：必须用 **（客）{客队} vs {主队}（±线）** 展示盘口（线与数据包 `rfsf_home.line` 一致：负=主队让，正=主队受让）；**禁止**「某队让分(线)」单句。让分主胜=主队赢盘（分差+线>0），让分主负=客队赢盘。
- 写入后 **`git add output/ && git commit && git push`** 到与后端 **`NBA_ANALYSIS_GIT_ROOT`** 相同的远端（需已配置凭证）；便于分析服务在本地对该仓库 **`git pull`** 后调用 **「同步产物」** 把 `parlay_recommendation.json` 写入数据库并在管理端展示。若环境无写权限，至少将上述 JSON **完整粘贴**给用户。

## 安全

仅处理用户明确给出的 `run_id` 与日期路径，不要遍历或修改其他 `runs/` 目录。

---

## 若已放入 `~/.openclaw/skills/nba-parlay-steps/SKILL.md` 但客户端里仍不显示

1. **技能由 Gateway 加载，不是本机 App 扫盘**  
   macOS App 通过 Gateway 的 **`skills.status`** 拉列表。`SKILL.md` 必须存在于 **运行 Gateway 进程的那台机器**上的 `~/.openclaw/skills/`（或该主机的 workspace `skills/`）。若 App 连接的是 **远程 / Docker 里的 Gateway**，在你笔记本家目录拷文件 **无效**，要把同一目录拷到 **Gateway 容器/服务器**对应用户主目录或配置的技能路径。

2. **改完后重启 Gateway**  
   复制/修改 skill 后需 **重启 Gateway 服务**（或开启配置里的 `skills.load.watch` 并等待刷新），否则列表不更新。

3. **`requires.bins: git` 不满足时会被隐藏**  
   Gateway 环境若 **PATH 里没有 `git`**（常见于精简容器），该 skill 可能被标为不可用或不出现在「可用」列表。在 **Gateway 主机**上执行 `which git`；若缺失则安装 git，或临时去掉 frontmatter 里 `metadata.openclaw.requires` 做排查（仍须在执行时自备 git）。

4. **配置 allowlist**  
   若 `~/.openclaw/openclaw.json`（或 Gateway 配置）里对 skills 做了 **allowlist / 禁用**，未列出的自定义 skill 可能不展示。检查 `skills.entries` 等项。

5. **自检命令**（在装 Gateway 的机器上，若已安装 CLI）  
   `openclaw skills list` 或查阅 Gateway 日志里 skill 加载报错（YAML 解析失败等）。
