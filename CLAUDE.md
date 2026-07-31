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

### 自动化验证（headless Edge）

这台机器上没有 node / python，但 Edge 能跑 headless，足够做真实的运行时测试。做法是**把测试装置注入 index.html 的副本**（不污染仓库）：顶层 `let`/`const` 声明在同一个文档的其它 classic script 里可见，所以注入的脚本能直接调 `S` / `render()` / `merge()` 等。

```powershell
$edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
# 1. 读 index.html，把 </body> 换成 <pre id="testout"></pre> + 断言脚本，另存到临时目录
# 2. 跑：
Start-Process $edge -ArgumentList '--headless=new','--disable-gpu','--no-sandbox',
  "--user-data-dir=$tmp\profile",'--allow-file-access-from-files',
  '--virtual-time-budget=8000','--dump-dom',"file:///$tmp/test.html" `
  -Wait -NoNewWindow -RedirectStandardOutput "$tmp\dom.txt"
# 3. 从 dom.txt 里正则抠出 <pre id="testout"> 的内容
```

坑：
- **必须用 `Start-Process -RedirectStandardOutput`**，直接 `& $edge ... > file` 拿不到输出。
- `--window-size` 在这个 Edge 版本里**不生效**，视口恒为 477px。要验证窄屏版式，注入 `<style>#app{max-width:360px !important}</style>` 再比较各元素的 `getBoundingClientRect().right` 和 `#app` 的右边界。
- `.chips` 本来就是横向滚动的，它「溢出」是设计如此，不是 bug。
- 截图用 `--screenshot=<path>`；图片会按 `--window-size` 裁剪而不是缩放，所以窄图会**假装**成内容被切掉，别被骗。

最近一次全量验证：51 项断言全通过，覆盖计划打勾、墓碑、merge、migrate、三层 sheet 导航、曲线渲染、录入三条路径。装置本身没有入库（属于一次性工具），要复现照上面重建即可。

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
| `训练计划` | `curPlan/planDay/planStat/doneRec/lastWt/togglePlanItem` |
| `训练` | 周视图、当日计划打勾、动作编辑 sheet |
| `分化管理` | 三层 sheet：`planSheet` → `planEditSheet` → `planDaySheet` |
| `趋势` | 近 14 天条形图 |
| `动作进展` | `exSessions/metricOf/curveSVG/progressCard` |
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
let draft   // 正在编辑的分化副本，保存前不碰 S.plans
let trendEx // 进展曲线当前选中的动作名
```

改完 `S` 之后的标准三连：`save(); render(); queueSync();`

### 数据模型（`S`，schema `v: 3`）

```js
{
  v: 3,
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

  plans: [{ id, name, ts,
    days: [ { name: "推", items: [{ name, sets, reps, wt }] }, … ]  // 恒为 7 项，周一..周日
  }],                            // items 为空 = 休息日
  activePlan: "<plan id>",       // 当前生效的分化，跟设置一起 LWW

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
| `entries` / `workouts` / `plans` | 按 `id` 合并，同 id 取 `ts` 大的；再用 `tomb` 过滤掉「删除时间晚于记录时间」的 |
| `weights` / `lib` | 按 key 逐条 last-write-wins（比 `ts`） |
| 设置（weight/kc/kp/kf/activePlan） | 整体按 `settingsTs` last-write-wins；合并后若选中的分化已不存在会自动清空 |
| `tomb` | 取并集，取较晚的时间戳；剪掉 120 天前的 |

触发点：`queueSync()`（4 秒防抖）、页面重新可见、`online` 事件、设置页手动按钮。`#syncdot` 是右上角状态灯（灰=闲 / 橙=同步中 / 绿=成功 / 红=失败）。

同步失败**静默**（`syncNow(true)`），只有手动触发才弹 toast。

### 饮食录入的三条路径

1. **快速命中**：输入「鸡胸肉 150」这类「名字+克数」，`quickParse()` 在 `S.lib` 里模糊匹配到就直接换算 → 走 `review()` 确认。
2. **粘贴 JSON**：库里没有的食物，在 claude.ai 用 `SYS`（设置页可一键复制）建的项目里查，把返回的 JSON 粘回输入框 → 应走 `pasteSheet()`。
3. **手动**：点「手动」按钮，直接填三大营养素克数（看包装营养表最快）→ `manualSheet()`。

确认保存时会把该食物的每 100g 数值**写回 `S.lib`**，下次就能走路径 1。

### 训练计划怎么工作

核心设计：**「完成」不是一个字段，而是「当天存在同名的 `workouts` 记录」**（`doneRec(date, name)`）。

这么做的好处，改动这块前务必理解：

- `workouts` 仍是训练历史的唯一真相，容量统计、CSV 导出、进展曲线、merge 全都不用动。
- 手动记的和打勾记的是同一份数据 —— 你自己加一条「深蹲」，计划里的深蹲会自动变成已完成。
- 打勾 = `push` 一条 `workouts`；取消打勾 = `removeRec`（写墓碑），所以多端同步天然一致。

配套细节：

- 打勾时组数/次数取计划值，重量取 `it.wt || lastWt(name, date)` —— 也就是**你上次练这个动作的重量**，做渐进超负荷时不用每次重填。
- 分化按**星期**铺（`dowIndex()` 周一=0），不是按 N 天循环。7 个槽位，`items` 为空即休息日。
- 当天记了但不在计划里的动作，归到「计划外」分组单独列。
- 编辑分化时改的是 `draft`（`S.plans` 里那条的深拷贝），只有点「保存」才写回。`closeSheet()` 会清掉 `draft`。
- `planDaySheet` 的 `grab()` **故意不过滤空名字的行**，否则删除按钮的下标会跟渲染时的下标错位；过滤只在离开该页和 `savePlan()` 时做。

### 进展曲线

`exSessions(name)` 把同一天同一动作的多条记录并成一次「训练课」，然后 `metricOf()` 按数据形态自动选指标：

| 条件 | 指标 |
|---|---|
| 有重量 | 预估 1RM（Epley：`wt × (1 + reps/30)`） |
| 徒手但有次数 | 总次数 |
| 只有时长 | 时长 |
| 其它 | 组数 |

图是手写的内联 SVG（`curveSVG()`），没有任何图表库：单序列所以不需要图例，标题即系列名；只直接标注首尾两点的值，中间的点靠点击更新下方读数；每个点额外叠一个 `r=13` 的透明圆做触摸目标。下方的「最近记录」列表同时充当非图形的可读回退。

## 已知问题

> v2 重构（`c4b2e6b`）曾把 `review()` / `pasteSheet()` / `copyText()` 删掉但留下了调用点，导致除「手动」外的所有录入路径 `ReferenceError`。已在 `2026-07-31.2` 修复：`review/drawItems/commit` 从 v1 取回，`parseItems/pasteSheet/copyText/legacyCopy` 为新写。改 `submit()` 那条链路时注意别再断。

现存的坑：

- **食物库删除会被同步复活。** `showLib()` 删条目只是 `delete S.lib[name]`，没有墓碑；下次同步时远端那条会被 merge 回来。要真删得给 `lib` 也加墓碑机制。
- **`KEY = "macros_v1"` 但 schema 已经是 `v: 3`。** 键名没跟着升，是历史遗留，别去改（改了会丢用户数据）。版本迁移靠 `migrate()` 里的 `o.v < 2` / `o.v < 3` 分支。
- **同名动作只会匹配到一条计划项。** `doneRec()` 取第一条同名记录；同一天同一动作记了两次，第二条会落到「计划外」分组。统计和曲线不受影响（`exSessions` 会把它们并起来）。
- **`quickParse()` 的模糊匹配靠 `Object.keys()` 顺序**，第一个 `includes` 命中就赢，结果不稳定。
- **编辑已有饮食记录不能改日期和餐次**（`manualSheet(rec)` 只改 name/g/c/p/f）。
- **`removeRec()` 自己不 render**，所有调用点都得手动跟一个 `render()`。
- `migrate()` 会主动 `delete s.apiKey / s.model / s.mode` —— v1 的遗留字段，别再用这些名字。

## 改代码时的约定

- **每次改动要顺手更新 `APP_VERSION`**（目前 `"2026-07-31.3"`，用日期串，同一天多次发布加 `.N`）。设置页「检查更新」是 `location.replace(pathname + "?u=" + Date.now())` 绕缓存重载，用户靠版本号确认自己刷到新版了。
- 新增持久化字段：在 `blank()` 里加默认值，在 `migrate()` 里处理老数据，在 `syncPayload()` 里决定要不要同步，在 `merge()` 里定义合并策略。**四个地方都要过一遍**，漏一个就会出现「同步后字段消失」。
- 新增记录类实体：必须有 `id`（用 `newId()`）和 `ts`，删除走 `removeRec()` 以写墓碑。
- 颜色只用 `:root` 里的 CSS 变量（`--carb` 橙 / `--prot` 青 / `--fat` 紫 / `--lift` 蓝 / `--over` 红 / `--ok` 绿），不要写死色值。数字一律用 `--mono` 字体加 `font-variant-numeric: tabular-nums`。
- 移动端安全区：新增贴边容器记得带 `env(safe-area-inset-*)`。但**清空内容 ≠ 元素消失** —— `#composer` 在非「今日」页只被 `innerHTML = ""`，它的边框和安全区内边距会在 tabbar 上方留一条约 50px 的空白（iPhone 上尤其明显）。靠 `#composer:empty{display:none}` 收掉，别删了那条规则。
- **复用别处的 class 时先看选择器有没有被限定父级。** 比如 `.ex1` 的排版规则写成 `.exrow .ex1 b{…}`，在新的 `.prow` 里复用 `.ex1` 就完全不生效（表现是名字和明细挤成一行、`<i>` 变斜体）。新增容器时把它加进选择器列表，别复制一份样式。
- 提交信息沿用中文短句风格（`每日营养摄入追踪表v3`）。

## 路线图

营养侧和训练侧的主体都已完成：

- ✅ 营养：三大营养素目标/剩余、四餐分组、食物库、趋势、CSV
- ✅ 训练记录：按日记录、周视图、组数/容量汇总、CSV
- ✅ 训练计划：分化模板（3 个内置预设）、按星期铺开、打勾完成、周完成度、按动作的进展曲线

还没做、想做可以从这里接：

- **渐进超负荷提示** —— 数据已经够了（`exSessions` + `lastWt`），可以在打勾时提示「上次 4×6 @60，这次试 62.5」
- **分化按 N 天循环**而不是绑星期（现在 `planDay()` 直接用 `dowIndex()`，改这里要同时改周视图的呈现）
- **组级记录** —— 现在一个动作一条记录，练到一半改重量只能记平均；要精确得让 `workouts` 带一个 `sets:[{reps,wt}]` 数组（注意四处同步 + `volOf` + CSV）
- **休息计时器**、**动作示范图** —— 都需要额外资源，跟「单文件零依赖」的约束冲突，想清楚再做
