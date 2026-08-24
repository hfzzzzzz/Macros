# CLAUDE.md

给未来在这个仓库里工作的 Claude Code / 开发者看的说明。

## 项目是什么

**Macros** —— 一个自用的「每日营养摄入追踪 + 健身训练计划」单页应用。

- 形态：**营养摄入追踪表** + **健身训练计划表**，两侧主体都已完成，剩下的想法见 [路线图](#路线图)。
- 使用场景：手机为主（iOS Safari 添加到主屏，全屏 PWA 风格），电脑为辅。
- 部署：GitHub Pages，仓库 `hfzzzzzz/Macros`，`main` 分支根目录的 `index.html` 即线上版本。**push 就是发布**。

## 仓库结构

```
index.html   ← 整个应用（HTML + CSS + JS 全部内联，约 1950 行）
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

**装置本身不入库**，写在临时目录里，会被系统清理掉 —— 已经丢过一次。所以它只是一次性工具，别指望它一直在；每次动大功能时照上面重建一份针对性的即可。

历史上跑过的五轮：

- `2026-08-03`（已丢失）：114 项，覆盖计划打勾、墓碑、merge、migrate、三层 sheet 导航、曲线渲染、录入三条路径、食物库清空/导入/不复活、手动录入入库、g/ml 单位换算、常规摄入。
- `2026-08-03.2`：42 项，专攻体重/维度 —— v5→v6 迁移逐字段核对、录入与删除、序列与曲线、三张图各自的点击派发、`meas` 的 LWW 合并；外加一轮冒烟（四个 tab 渲染 + 五个 sheet 能开 + g/ml 换算 + 食物库导入）确认别处没被改坏。
- `2026-08-03.3`：58 项，专攻训练侧模块化 —— 六个部位的动作清单逐条核对、预设由部位生成且不污染常量、部位选择器展开/收起/整组加入/不重复、稳定重量的读取与打勾优先级、容量在六处显示中确实消失；外加一轮冒烟。
- `2026-08-03.4`：41 项，专攻目标体重与体重记录解耦 —— 改记录不动目标、v6→v7 迁移后目标一模一样、两处录入互不写对方、设置页两张卡、`weight` 与 `weights` 各走各的合并策略；外加一轮冒烟。
- `2026-08-03.5`（当前）：39 项，专攻每日锻炼输出 —— 文本清单的编号/组次/有氧/备注/空日期各种形态、当天 CSV 能被自家 parseCSV 读回来（含逗号引号的动作名）、两处入口的出现条件、`navigator.share` 有无时的降级；外加一轮冒烟。

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
| 常量 | `KCAL`、餐次、星期、`baseOf()` 单位基准、`PARTS` 动作目录、内置分化预设 |
| `state` | `blank()` / `migrate()` / `load()` / `save()` |
| `date helpers` | 日期全部用 `"YYYY-MM-DD"` 字符串，不用 Date 对象传递 |
| `computed` | `lastWeight()` / `targets()` / `eatenOn()` / `exOn()` / `workWt()` |
| `render` | `render()` 总调度 + 四个 tab 的渲染函数 |
| `训练计划` | `curPlan/planDay/planStat/doneRec/workWt/togglePlanItem` |
| `部位选择器` | `mountPartPicker()` —— 两级 chip，加动作和排分化共用 |
| `训练` | 周视图、当日计划打勾、动作编辑 sheet |
| `每日锻炼输出` | `dayText()` / `dayCSV()` / `exportDaySheet()` |
| `分化管理` | 三层 sheet：`planSheet` → `planEditSheet` → `planDaySheet` |
| `趋势` | 近 14 天条形图 |
| `体重与维度` | `weightSeries/measSeries/seriesBlock/bodyCards/bodySheet` |
| `动作进展` | `exSessions/metricOf/curveSVG/progressCard` |
| `设置` | 饮食目标（目标体重+系数）、身体记录入口、同步配置、数据导入导出 |
| `records / tombstones` | `newId()` / `removeRec()` |
| `merge` | 本地 ↔ 远端的合并算法 |
| `Gist sync` | GitHub API 封装、创建 gist、`syncNow()`、`queueSync()` |
| `常规摄入` | `routineItems/applyRoutine/routineSheet/routineEditSheet` |
| `食物库` | `showLib()` / `dropLib()` / `exportLib()` / `parseCSV()` / `importLibText()` / `libImportSheet()` |
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
let draft   // 正在编辑的分化 / 常规搭配副本，保存前不碰 S.plans / S.routines
let trendEx // 进展曲线当前选中的动作名
let trendMeas// 维度曲线当前选中的项目 arm/chest/waist/hip
```

改完 `S` 之后的标准三连：`save(); render(); queueSync();`

### 数据模型（`S`，schema `v: 7`）

```js
{
  v: 7,
  weight: 70,                    // 目标体重：只用来算营养目标，跟 weights 无关
  kc: 3, kp: 1.7, kf: 1,         // 碳水/蛋白/脂肪，g per kg 体重
  settingsTs: 0,                 // 上述设置整体的 last-write-wins 时间戳

  entries: [{ id, date, meal, name, g, u, c, p, f, ts }],
              // g 是分量数值，u 是它的单位（"g" / "ml"）
              // c/p/f 是这一条的绝对克数（不是每份）
  workouts: [{ id, date, name, sets, reps, wt, dur, note, ts }],
              // 用不到的字段存 0 / ""；wt 就是那天做组用的重量
  weights:  { "2026-07-31": { v: 72.5, ts } },              // kg
  meas:     { "2026-07-31": { arm, chest, waist, hip, ts } },// cm，字段可缺；全缺就不存这天
  lib:      { "鸡胸肉":  { c: 0,   p: 23.1, f: 1.9,  u: "g",  ts },   // 每 100 g
              "椰子奶":  { c: 4.5, p: 1.3,  f: 11.5, u: "ml", ts } }, // 每 250 ml
  tomb:     { "<被删记录的 id>": ts },                      // 墓碑，120 天后清理
  libTomb:  { "鸡胸肉": ts },                               // 食物库的墓碑，同上

  plans: [{ id, name, ts,
    days: [ { name: "推", items: [{ name, sets, reps, wt }] }, … ]  // 恒为 7 项，周一..周日
  }],                            // items 为空 = 休息日
  activePlan: "<plan id>",       // 当前生效的分化，跟设置一起 LWW

  routines: [{ id, name, meal, ts,
    items: [{ name, q }] }],     // 常规搭配；q 的单位取 lib 里那条的 u，不单独存

  sync: { token, gistId, last }  // ← 仅本机，不进 syncPayload()
}
```

计算规则：
- 目标 = `S.weight` × 系数，跟日期无关。**体重记录不参与计算**，见下面一节。
- 热量 = `c*4 + p*4 + f*9`（`KCAL` 常量）。

### 目标体重 vs 体重记录（两回事，别再合起来）

这两个东西名字像，但**刻意是分开的**：

| | 存在哪 | 谁在用 | 谁能改 |
|---|---|---|---|
| **目标体重** | `S.weight`（单个数） | `targets()` 算三大营养素目标 | 只有设置页「饮食目标」里那个输入框 |
| **体重记录** | `S.weights`（日期 → 体重） | 趋势页的体重曲线 | 只有 `bodySheet()` |

- **`bodySheet()` 绝不写 `S.weight`，设置页那个框也绝不写 `S.weights`。** 以前是耦合的（记一次体重就顺手改了目标，历史日期还会按当日体重反算历史目标），`2026-08-03.4` 拆开了。别再加回任何一边的联动。
- 要让目标跟上最新体重，得在设置页**主动点**「目标体重改用 X kg」——这个按钮只在两者相差超过 0.05 kg 时出现。
- `targets()` **不接受日期参数**了，所有日期的目标都一样。趋势页的达成率、CSV 的目标列都跟着变成常量。
- `lastWeight()` 只是给设置页做参考显示和上面那个按钮用的，别拿它去算目标。
- 迁移 `v6 → v7` 时用老算法取了一次当时的值写进 `S.weight`，所以**升级前后每日目标完全一致**，用户不会突然发现目标变了。

### 单位：一份 = 100 g 或 250 ml

食物库不按「每 100」存，按**一份**存，`baseOf(u)` 给出一份多大：固体 `g` → 100，液体 `ml` → 250。所以所有换算都是 `qty / base`，**不要再写死 `/100`**。

- `S.lib` 里每条带 `u`；老数据没有这个字段，`migrate()` 一律补成 `"g"`，数值一个不动 —— 升级前后固体食物的行为完全一样。
- 确认清单里的条目字段是 `{ name, qty, u, base, cb, pb, fb }`（`cb/pb/fb` = 每份的克数）。曾经叫 `grams/c100/p100/f100`，跟着单位一起改了名，别再按老名字找。
- `libItem(name, qty)` 是从食物库造这种条目的唯一入口，单位和 base 都跟着库走，新代码用它就不会算错。
- Claude 返回的 JSON 仍然是**每 100 g / 每 100 ml**（对模型更自然），`parseItems()` 在读进来时给液体乘 2.5 折算成每 250 ml。`amount`+`unit` 是新字段，老提示词只有 `grams` 也照样能用。
- 一份里装不下超过一份重量的营养素，所以导入的上限是 `baseOf(u) * 1.005`，液体那条自然放宽到 251.25。

### 同步协议

两端填**同一个 GitHub Token（需 gist 权限）+ 同一个 Gist ID**，数据镜像到一个 secret gist 里的 `macros.json`。

`syncNow()` 的流程是 **拉 → 合并 → 全量写回**：

1. GET gist，解析 `macros.json`
2. `merge(local, remote)`
3. PATCH 回去（整份覆盖）

`merge()` 的规则：

| 字段 | 策略 |
|---|---|
| `entries` / `workouts` / `plans` / `routines` | 按 `id` 合并，同 id 取 `ts` 大的；再用 `tomb` 过滤掉「删除时间晚于记录时间」的 |
| `weights` / `meas` / `lib` | 按 key（日期 / 食物名）逐条 last-write-wins（比 `ts`）；`lib` 再用 `libTomb` 过滤一遍 |
| `tomb` / `libTomb` | 取并集，取较晚的时间戳；剪掉 120 天前的 |
| 设置（weight/kc/kp/kf/activePlan） | 整体按 `settingsTs` last-write-wins；合并后若选中的分化已不存在会自动清空 |

触发点：`queueSync()`（4 秒防抖）、页面重新可见、`online` 事件、设置页手动按钮。`#syncdot` 是右上角状态灯（灰=闲 / 橙=同步中 / 绿=成功 / 红=失败）。

同步失败**静默**（`syncNow(true)`），只有手动触发才弹 toast。

### 饮食录入的三条路径

1. **快速命中**：输入「鸡胸肉 150」这类「名字+克数」，`quickParse()` 在 `S.lib` 里模糊匹配到就直接换算 → 走 `review()` 确认。
2. **粘贴 JSON**：库里没有的食物，在 claude.ai 用 `SYS`（设置页可一键复制）建的项目里查，把返回的 JSON 粘回输入框 → 应走 `pasteSheet()`。
3. **手动**：点「手动」按钮，直接填三大营养素克数（看包装营养表最快）→ `manualSheet()`。

**三条路径都会把该食物每一份的数值写回 `S.lib`**，下次就能走路径 1；`lib` 在 `syncPayload()` 里，所以也就顺带同步到别的设备了。

路径 1、2 由 `commit()` 无条件写回。路径 3 多一个开关（`#m_lib`，默认开）：库里按一份存，所以**必须同时有名称和分量**才能换算，`perBase()` 拿不到就返回 null、开关自动禁用并提示原因。像「外卖」这种一次性的记录，把开关关掉就只记饮食不进库。手动录入的分量旁边有 g / ml 选择器；名字命中食物库时单位以库里的为准。

`S.lib` 出厂为空 —— 没有任何内置种子食物，全靠你自己录入或导入。

### 食物库的批量维护

「设置 → 管理食物库」里有导入 / 导出 / 清空。CSV 五列，数值按**一份**填：

```
食物名称,单位,每份碳水,每份蛋白,每份脂肪
鸡胸肉,g,0,23.1,1.9
椰子奶,ml,4.5,1.3,11.5
"牛肉,瘦",g,0,22.6,5
```

- `parseCSV()` 是手写的，支持引号包裹、`""` 转义、CRLF、开头 BOM。别换成 `split(",")`，食物名里有逗号就废了。
- **老的四列格式（没有单位列）仍然认**，按固体处理。判断靠「第二列能不能 parseFloat 成数字」：不能就是单位列。
- `importLibText(text, overwrite)` 会跳过表头和空行，拒绝负数和超过一份重量的行，并**撤掉该名字的墓碑**。`overwrite=false` 时库里已有的同名食物整条跳过，数值不动；导入结束按「新增 / 更新 / 跳过 / 无效」分别报数，不会静默吞掉任何一行。
- 导入入口同时支持粘贴和选文件 —— 手机上粘贴好用得多，别只留文件选择。
- 「清空」要点两次（4 秒内），第一次只是把按钮变成「再点一次确认」。

**清空之所以能生效，靠的是 `dropLib()` 写 `libTomb`**，否则下次同步远端会把整库捞回来。别改成裸的 `delete S.lib[name]`。

（曾经还有第二个坑：`migrate()` 里有一段「lib 空了就重新播种内置的 26 条」。`DEFAULT_LIB` 已在 `2026-07-31.6` 整个删掉，那段逻辑也一并没了。不要再加回任何形式的内置种子。）

### 常规摄入（每天固定吃的那几样）

`S.routines` 存搭配，一组 = 名称 + 记入哪一餐 + 若干「食物 + 分量」。应用时 `applyRoutine()` 把它铺成待确认清单走 `review()`，**不直接写库**，所以今天少吃点可以当场改分量。

- 分量的单位不存在 routine 里，`unitOf(name)` 从 `S.lib` 现取 —— 改了食物库的单位，搭配跟着变，不会对不上。
- 库里没有的食物在编辑器里标红，应用时跳过并 toast 提示，不会静默丢。
- 从食物库点着加时，默认分量就是一份（固体 100 g / 液体 250 ml），改一下就行。
- 入口有两个：今日页 composer 顶上的 ⚡ chip 行（只在有搭配时才渲染，没有就不占地方），和「设置 → 常规摄入」。
- 编辑走 `draft`，跟分化编辑器同一套；`routineEditSheet` 的 `grab()` 同样**不过滤空行**，否则删除按钮下标错位。

### 动作目录按部位模块化

`PARTS` 是训练侧唯一的动作来源：六个部位（胸 / 背 / 肩 / 手臂 / 腿部 / 有氧+核心），每个部位一串动作名。**想改动作清单只动这一处**，内置分化、部位选择器、CSV 的部位列全是照着它铺的。

- `PART_OF` 是「动作名 → 部位」的反查表，同名动作（比如「反向山羊挺身卷腹」出现在多个部位）归到**第一个**出现的部位。
- `PRESETS` 由 `PARTS.map(partDay)` 生成，不再手写：六分化（一天一个部位）和五分化（练五休二）。默认 3×10、重量留 0。
- `mountPartPicker(id, onPick, onAll)` 是两级 chip 选择器，挂在一个容器 id 上。展开/收起**只重画自己**，不碰同一张 sheet 上已填好的表单 —— 这是它没写成整体重渲染的原因。`onAll` 给了就多一个「整组加入」按钮。
- 加动作（`exSheet`）和排分化（`planDaySheet`）共用它。加动作时选中会顺带填上 3×10 和该动作的稳定重量。
- 「最近用过」的 chip 只列**不在目录里**的动作名，避免和目录重复。

### 稳定做组的重量

`workWt(name, before)` = 该动作最近一次真正上过重量（`wt > 0`）的那条记录的重量；`before` 给了就只看更早的。

- 分化里每个动作可以存一个 `wt`（`planDaySheet` 里那个 kg 输入框），这就是你稳定做组的重量。
- 打勾时取 `it.wt || workWt(name, date)` —— 优先用计划里设的，没设就带出上次的。
- 训练页每行右侧显示这个重量：做完了显示实做值（亮色），没做显示计划值或历史值（`.vol.ghost` 暗色）作提示。
- 进展曲线的默认指标就是它（`metricOf` 的 `top`）。**曾经是 Epley 预估 1RM 和训练容量，都已经去掉了** —— 别再加回来。

### 每日锻炼输出 / WHOOP（别再去试直接导入了）

`exportDaySheet(date)` 把当天练的排成一份纯文本清单（`dayText()`），外加当天 CSV（`dayCSV()`）。入口两个：训练页每天右上的「输出」（只在当天有记录时出现）、今日页的训练汇总行。

**为什么不做直接导入 WHOOP** —— 查证过，三条路都堵死：

1. **官方 API 是只读的。** `developer.whoop.com` 的 changelog 到 `2025-11-01` 为止没有任何 POST/写入端点。
2. **Strength Trainer 的明细压根不在 API 里。** `2024-05-01` 那条更新只是让 Strength Trainer 作为一种活动类型出现在 `/workout` 里；组数、次数、重量既读不到也写不进。
3. **逆向的私有接口对这个 App 也没用。** 静态页从 GitHub Pages 跨域调 `api.prod.whoop.com` 会被 CORS 拦掉，而且要存 WHOOP 凭据 —— 跟[设计约束](#设计约束)的第 2、3 条直接冲突。

所以这里只做「让手动录入尽量快」：清单顺序就是 WHOOP 里的录入顺序，`navigator.share` 可用时给分享按钮（iOS 上直接开系统分享面板），不可用就只留复制。**注意分享被用户取消时 `navigator.share` 会 reject，要吞掉，别弹失败提示。**

哪天 WHOOP 真开了写入端点，再回来看这一节。

### 训练计划怎么工作

核心设计：**「完成」不是一个字段，而是「当天存在同名的 `workouts` 记录」**（`doneRec(date, name)`）。

这么做的好处，改动这块前务必理解：

- `workouts` 仍是训练历史的唯一真相，组数统计、CSV 导出、进展曲线、merge 全都不用动。
- 手动记的和打勾记的是同一份数据 —— 你自己加一条「深蹲」，计划里的深蹲会自动变成已完成。
- 打勾 = `push` 一条 `workouts`；取消打勾 = `removeRec`（写墓碑），所以多端同步天然一致。

配套细节：

- 打勾时组数/次数取计划值，重量取 `it.wt || lastWt(name, date)` —— 也就是**你上次练这个动作的重量**，做渐进超负荷时不用每次重填。
- 分化按**星期**铺（`dowIndex()` 周一=0），不是按 N 天循环。7 个槽位，`items` 为空即休息日。
- 当天记了但不在计划里的动作，归到「计划外」分组单独列。
- 编辑分化时改的是 `draft`（`S.plans` 里那条的深拷贝），只有点「保存」才写回。`closeSheet()` 会清掉 `draft`。
- `planDaySheet` 的 `grab()` **故意不过滤空名字的行**，否则删除按钮的下标会跟渲染时的下标错位；过滤只在离开该页和 `savePlan()` 时做。

### 体重与身体维度

`S.weights` 和 `S.meas` 都是「按日期存一条」，没有 id、没有墓碑，合并就是按日期逐条比 `ts`。删一天 = 把两个 map 里那天都 `delete` 掉。

- 维度只有四项（`MEAS`：臂围 / 胸围 / 腰围 / 臀围），全部按厘米。想加项目就往 `MEAS` 里加一条，录入表单、chip、曲线都是照着它铺的，别的地方不用改。
- `bodySheet(date)` 一次记体重 + 四围，**留空的项目不写也不覆盖**；一项都没填会被拦下。四围全空时不会留下空的 `meas` 记录（`migrate()` 也会顺手清掉历史上的空记录）。
- 体重仍然是营养目标的输入（`weightOn()`），所以在这里改历史体重会改历史目标 —— 跟设置页那个输入框是同一份数据。
- 三张曲线（体重 / 维度 / 动作进展）**共用 `curveSVG()` 和同一个 `[data-pt]` 事件**，靠 `data-pt="<key>|<i>"` 的 key 区分（`w` / `m` / `ex`），各自更新 `#wread` / `#mread` / `#exread`。再加图就沿用这个约定。
- 变化量**刻意不判好坏**：减脂时体重降是好事，增肌时又反过来，App 不知道你的目标，所以不用 `.ok`/`.bad` 上色。动作进展那张图涨=好没歧义，保留着上色。

### 进展曲线

`exSessions(name)` 把同一天同一动作的多条记录并成一次「训练课」，然后 `metricOf()` 按数据形态自动选指标：

| 条件 | 指标 |
|---|---|
| 有重量 | 稳定做组重量（当天那条记录的 `wt`） |
| 徒手但有次数 | 总次数 |
| 只有时长 | 时长 |
| 其它 | 组数 |

图是手写的内联 SVG（`curveSVG()`），没有任何图表库：单序列所以不需要图例，标题即系列名；只直接标注首尾两点的值，中间的点靠点击更新下方读数；每个点额外叠一个 `r=13` 的透明圆做触摸目标。下方的「最近记录」列表同时充当非图形的可读回退。

## 已知问题

> v2 重构（`c4b2e6b`）曾把 `review()` / `pasteSheet()` / `copyText()` 删掉但留下了调用点，导致除「手动」外的所有录入路径 `ReferenceError`。已在 `2026-07-31.2` 修复：`review/drawItems/commit` 从 v1 取回，`parseItems/pasteSheet/copyText/legacyCopy` 为新写。改 `submit()` 那条链路时注意别再断。

现存的坑：

- **`KEY = "macros_v1"` 但 schema 已经是 `v: 4`。** 键名没跟着升，是历史遗留，别去改（改了会丢用户数据）。版本迁移靠 `migrate()` 里的 `o.v < 2` / `< 3` / `< 4` 分支。
- **食物库墓碑 120 天后会被剪掉。** 剪掉之后，如果某个设备还留着老的 `lib` 条目（`ts: 0` 的种子尤其），它可能重新出现。跟 `tomb` 是同一个取舍，日常用不到这个时间尺度。
- **同名动作只会匹配到一条计划项。** `doneRec()` 取第一条同名记录；同一天同一动作记了两次，第二条会落到「计划外」分组。统计和曲线不受影响（`exSessions` 会把它们并起来）。
- **`quickParse()` 的模糊匹配靠 `Object.keys()` 顺序**，第一个 `includes` 命中就赢，结果不稳定。
- **编辑已有饮食记录不能改日期和餐次**（`manualSheet(rec)` 只改 name/g/c/p/f）。
- **`removeRec()` 自己不 render**，所有调用点都得手动跟一个 `render()`。
- `migrate()` 会主动 `delete s.apiKey / s.model / s.mode` —— v1 的遗留字段，别再用这些名字。

## 改代码时的约定

- **每次改动要顺手更新 `APP_VERSION`**（目前 `"2026-08-03.5"`，用日期串，同一天多次发布加 `.N`）。设置页「检查更新」是 `location.replace(pathname + "?u=" + Date.now())` 绕缓存重载，用户靠版本号确认自己刷到新版了。
- 新增持久化字段：在 `blank()` 里加默认值，在 `migrate()` 里处理老数据，在 `syncPayload()` 里决定要不要同步，在 `merge()` 里定义合并策略。**四个地方都要过一遍**，漏一个就会出现「同步后字段消失」。
- 新增记录类实体：必须有 `id`（用 `newId()`）和 `ts`，删除走 `removeRec()` 以写墓碑。
- 颜色只用 `:root` 里的 CSS 变量（`--carb` 橙 / `--prot` 青 / `--fat` 紫 / `--lift` 蓝 / `--over` 红 / `--ok` 绿），不要写死色值。数字一律用 `--mono` 字体加 `font-variant-numeric: tabular-nums`。
- 移动端安全区：新增贴边容器记得带 `env(safe-area-inset-*)`。但**清空内容 ≠ 元素消失** —— `#composer` 在非「今日」页只被 `innerHTML = ""`，它的边框和安全区内边距会在 tabbar 上方留一条约 50px 的空白（iPhone 上尤其明显）。靠 `#composer:empty{display:none}` 收掉，别删了那条规则。
- **复用别处的 class 时先看选择器有没有被限定父级。** 比如 `.ex1` 的排版规则写成 `.exrow .ex1 b{…}`，在新的 `.prow` 里复用 `.ex1` 就完全不生效（表现是名字和明细挤成一行、`<i>` 变斜体）。新增容器时把它加进选择器列表，别复制一份样式。
- 提交信息沿用中文短句风格（`每日营养摄入追踪表v3`）。

## 路线图

营养侧和训练侧的主体都已完成：

- ✅ 营养：三大营养素目标/剩余、四餐分组、食物库、趋势、CSV
- ✅ 训练记录：按日记录、周视图、动作数/组数汇总、CSV
- ✅ 训练计划：分化模板（3 个内置预设）、按星期铺开、打勾完成、周完成度、按动作的进展曲线

还没做、想做可以从这里接：

- **渐进超负荷提示** —— 数据已经够了（`exSessions` + `lastWt`），可以在打勾时提示「上次 4×6 @60，这次试 62.5」
- **分化按 N 天循环**而不是绑星期（现在 `planDay()` 直接用 `dowIndex()`，改这里要同时改周视图的呈现）
- **组级记录** —— 现在一个动作一条记录，练到一半改重量只能记平均；要精确得让 `workouts` 带一个 `sets:[{reps,wt}]` 数组（注意四处同步 + `exSessions` + CSV）
- **休息计时器**、**动作示范图** —— 都需要额外资源，跟「单文件零依赖」的约束冲突，想清楚再做
