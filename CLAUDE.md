# CLAUDE.md

给未来在这个仓库里工作的 Claude Code / 开发者看的说明。

## 项目是什么

**Macros** —— 一个自用的「每日营养摄入追踪 + 健身训练记录」单页应用。

- 目标形态：**营养摄入追踪表** + **健身训练计划表**（训练侧目前只做到「记录」，还不是「计划」，见 [路线图](#路线图)）。
- 使用场景：手机为主（iOS Safari 添加到主屏，全屏 PWA 风格），电脑为辅。
- 部署：GitHub Pages，仓库 `hfzzzzzz/Macros`，`main` 分支根目录的 `index.html` 即线上版本。**push 就是发布**。

## 仓库结构

```
index.html   ← 整个应用（HTML + CSS + JS 全部内联，约 930 行）
README.md    ← 只有一行标题
CLAUDE.md    ← 本文件
```

**没有构建步骤、没有依赖、没有 package.json、没有测试框架。** 这是刻意的约束，见下面的[设计约束](#设计约束)。

### 怎么跑 / 怎么验证

直接用浏览器打开 [index.html](index.html) 即可，无需服务器（除非要测同步 —— GitHub API 走 fetch，`file://` 下 CORS 正常，可以工作）。

想起本地服务器：

```powershell
python -m http.server 8000    # 然后开 http://localhost:8000
```

**验证只能靠手动**：改完后打开页面，走一遍「记一条饮食 → 切到训练加一个动作 → 看趋势 → 设置里改系数」，确认没有 JS 报错（开控制台看）。目前没有任何自动化测试。

## 设计约束（改代码前先读）

这些不是偶然，是这个项目的核心取舍：

1. **单文件、零依赖、零构建。** 不要引入 npm、打包器、框架、CDN 脚本。用户要能把 `index.html` 直接丢到任何静态托管上，也要能在断网时打开。
2. **本地优先（local-first）。** 所有数据存 `localStorage`，键名 `macros_v1`。没有服务端，没有账号系统。同步是可选的增强，不是前提。
3. **密钥永不出本机。** GitHub Token 存在 `S.sync.token`，写进 localStorage，但 `syncPayload()` 刻意不包含它。改同步逻辑时**必须**保持这一点。
4. **不在 App 里调用 LLM API。** v1 曾内置 Anthropic API 调用（自带 API key，按 token 计费）。v2 起改为「在 Claude App 里查完，把 JSON 粘回来」，走用户订阅、不产生额外费用。不要把 API 调用加回去。
5. **手写 DOM 字符串 + `innerHTML`。** 没有虚拟 DOM。所有渲染函数返回 HTML 字符串塞进 `#scroll` 或 sheet，然后重新绑定事件。**任何用户输入拼进 HTML 前必须过 `esc()`**（食物名、动作名、备注都是用户可控的）。
6. **中文界面。** 所有面向用户的文案是简体中文，代码注释也是中文。保持一致。

## 架构

`index.html` 里 `<script>` 用注释横幅分成若干段，按顺序：

| 段落 | 职责 |
|---|---|
| 常量 / `DEFAULT_LIB` | `KCAL` 换算系数、餐次、星期、内置 26 条食物库种子 |
| `state` | `blank()` / `migrate()` / `load()` / `save()` |
| `date helpers` | 日期全部用 `"YYYY-MM-DD"` 字符串，不用 Date 对象传递 |
| `computed` | `weightOn()` / `targets()` / `eatenOn()` / `exOn()` / `volOf()` |
| `render` | `render()` 总调度 + 四个 tab 的渲染函数 |
| `训练` | 周视图、动作编辑 sheet |
| `趋势` | 近 14 天条形图 |
| `设置` | 体重系数、同步配置、数据导入导出 |
| `records / tombstones` | `newId()` / `removeRec()` |
| `merge` | 本地 ↔ 远端的合并算法 |
| `Gist sync` | GitHub API 封装、创建 gist、`syncNow()`、`queueSync()` |
| `食物库` | `showLib()` |
| `sheet` | 底部弹层的开关 |
| `饮食录入` | `submit()` / `quickParse()` / `SYS` 提示词 |
| `手动输入` | `manualSheet()` |
| `导入导出` | CSV / JSON |
| `boot` | tab 绑定、可见性变化触发同步、首次 render |

### 全局可变状态

```js
let S       // 全部持久化数据，见下方数据模型
let view    // "today" | "train" | "trend" | "set"
let cursor  // 今日页当前日期 "YYYY-MM-DD"
let wkCursor// 训练页当前周的周一
let meal    // 当前选中餐次
let pending // review sheet 里待确认的条目数组
```

改完 `S` 之后的标准三连：`save(); render(); queueSync();`

### 数据模型（`S`，schema `v: 2`）

```js
{
  v: 2,
  weight: 70,                    // 兜底体重，只在 weights 为空时用
  kc: 3, kp: 1.7, kf: 1,         // 碳水/蛋白/脂肪，g per kg 体重
  settingsTs: 0,                 // 上述设置整体的 last-write-wins 时间戳

  entries: [{ id, date, meal, name, g, c, p, f, ts }],
              // c/p/f 是这一条的绝对克数（不是每 100g）
  workouts: [{ id, date, name, sets, reps, wt, dur, note, ts }],
              // 用不到的字段存 0 / ""；容量 = sets*reps*wt
  weights:  { "2026-07-31": { v: 72.5, ts } },
  lib:      { "鸡胸肉": { c: 0, p: 23.1, f: 1.9, ts } },   // 每 100g
  tomb:     { "<被删记录的 id>": ts },                      // 墓碑，120 天后清理

  sync: { token, gistId, last }  // ← 仅本机，不进 syncPayload()
}
```

计算规则：
- 目标 = **当日体重** × 系数。`weightOn(date)` 取 `<= date` 的最近一条体重，所以改历史体重会改历史目标。
- 热量 = `c*4 + p*4 + f*9`（`KCAL` 常量）。
- 训练容量 = `sets × reps × wt`，任一为 0 则为 0。

### 同步协议

两端填**同一个 GitHub Token（需 gist 权限）+ 同一个 Gist ID**，数据镜像到一个 secret gist 里的 `macros.json`。

`syncNow()` 的流程是 **拉 → 合并 → 全量写回**：

1. GET gist，解析 `macros.json`
2. `merge(local, remote)`
3. PATCH 回去（整份覆盖）

`merge()` 的规则：

| 字段 | 策略 |
|---|---|
| `entries` / `workouts` | 按 `id` 合并，同 id 取 `ts` 大的；再用 `tomb` 过滤掉「删除时间晚于记录时间」的 |
| `weights` / `lib` | 按 key 逐条 last-write-wins（比 `ts`） |
| 设置（weight/kc/kp/kf） | 整体按 `settingsTs` last-write-wins |
| `tomb` | 取并集，取较晚的时间戳；剪掉 120 天前的 |

触发点：`queueSync()`（4 秒防抖）、页面重新可见、`online` 事件、设置页手动按钮。`#syncdot` 是右上角状态灯（灰=闲 / 橙=同步中 / 绿=成功 / 红=失败）。

同步失败**静默**（`syncNow(true)`），只有手动触发才弹 toast。

### 饮食录入的三条路径

1. **快速命中**：输入「鸡胸肉 150」这类「名字+克数」，`quickParse()` 在 `S.lib` 里模糊匹配到就直接换算 → 走 `review()` 确认。
2. **粘贴 JSON**：库里没有的食物，在 claude.ai 用 `SYS`（设置页可一键复制）建的项目里查，把返回的 JSON 粘回输入框 → 应走 `pasteSheet()`。
3. **手动**：点「手动」按钮，直接填三大营养素克数（看包装营养表最快）→ `manualSheet()`。

确认保存时会把该食物的每 100g 数值**写回 `S.lib`**，下次就能走路径 1。

## 已知问题

> v2 重构（`c4b2e6b`）曾把 `review()` / `pasteSheet()` / `copyText()` 删掉但留下了调用点，导致除「手动」外的所有录入路径 `ReferenceError`。已在 `2026-07-31.2` 修复：`review/drawItems/commit` 从 v1 取回，`parseItems/pasteSheet/copyText/legacyCopy` 为新写。改 `submit()` 那条链路时注意别再断。

现存的坑：

- **食物库删除会被同步复活。** `showLib()` 删条目只是 `delete S.lib[name]`，没有墓碑；下次同步时远端那条会被 merge 回来。要真删得给 `lib` 也加墓碑机制。
- **`KEY = "macros_v1"` 但 schema 是 `v: 2`。** 键名没跟着升，是历史遗留，别去改（改了会丢用户数据）。版本迁移靠 `migrate()` 里的 `o.v < 2` 分支。
- **`quickParse()` 的模糊匹配靠 `Object.keys()` 顺序**，第一个 `includes` 命中就赢，结果不稳定。
- **编辑已有饮食记录不能改日期和餐次**（`manualSheet(rec)` 只改 name/g/c/p/f）。
- **`removeRec()` 自己不 render**，所有调用点都得手动跟一个 `render()`。
- `migrate()` 会主动 `delete s.apiKey / s.model / s.mode` —— v1 的遗留字段，别再用这些名字。

## 改代码时的约定

- **每次改动要顺手更新 `APP_VERSION`**（目前 `"2026-07-31"`，用日期串）。设置页「检查更新」是 `location.replace(pathname + "?u=" + Date.now())` 绕缓存重载，用户靠版本号确认自己刷到新版了。
- 新增持久化字段：在 `blank()` 里加默认值，在 `migrate()` 里处理老数据，在 `syncPayload()` 里决定要不要同步，在 `merge()` 里定义合并策略。**四个地方都要过一遍**，漏一个就会出现「同步后字段消失」。
- 新增记录类实体：必须有 `id`（用 `newId()`）和 `ts`，删除走 `removeRec()` 以写墓碑。
- 颜色只用 `:root` 里的 CSS 变量（`--carb` 橙 / `--prot` 青 / `--fat` 紫 / `--lift` 蓝 / `--over` 红 / `--ok` 绿），不要写死色值。数字一律用 `--mono` 字体加 `font-variant-numeric: tabular-nums`。
- 移动端安全区：新增贴边容器记得带 `env(safe-area-inset-*)`。
- 提交信息沿用中文短句风格（`每日营养摄入追踪表v3`）。

## 路线图

用户的目标是「营养摄入追踪表 **+ 健身训练计划表**」，训练侧目前只完成了一半：

- ✅ 已有：按日记录动作、周视图、组数/容量汇总、CSV 导出
- ❌ 缺少「计划」：训练模板 / 分化（推拉腿等）、把模板套到某一周生成待做清单、完成打勾、按动作看重量进展曲线、渐进超负荷提示

做「计划」时的自然扩展点：在 `S` 里加 `plans`（模板）与 `workouts` 上的 `planId` / `done` 字段，训练页从「日志」改成「今日计划 + 已完成」两段，模板管理放设置页。注意上面「新增持久化字段」的四处同步。
