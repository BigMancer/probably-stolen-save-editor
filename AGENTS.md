# AGENTS.md — 项目记忆（深空当铺 存档修改器）

给继续维护这个项目的人 / Agent 看。**改动前先读「硬约束」一节。**

## 1. 项目是什么

**Probably Stolen**（中文名：深空当铺）的存档修改器。单文件纯前端：`存档修改器.html`
（≈77 KB，无依赖、无构建、离线运行）。用浏览器打开 → 拖入 `.es3` → 改 → 导出覆盖。

仓库内容：

| 文件 | 说明 |
|---|---|
| `存档修改器.html` | 整个修改器。解析器 / 导出器 / 全部 UI 都在里面 |
| `README.md` / `README.zh-CN.md` | 英文（默认）/ 中文说明 |
| `AGENTS.md` | 项目记忆（本文件） |
| `IDEA.md` | 最初的原始需求记录，保留 |
| `*.es3` | 游戏存档，个人数据，**已忽略**（`.gitignore`） |

游戏存档目录（Windows）：

```
%USERPROFILE%\AppData\LocalLow\Questing Goose Studio\Probably Stolen\
    save_1.es3        当前存档（最常用）
    save_0.es3        另一个槽位
    SaveFile.es3      很小的元数据
    saves_index.es3   存档列表索引
    Player.log / GameLog_*.txt
```

## 2. 存档格式：实测事实（不要凭印象）

`.es3` 是 Unity 风格的 JSON，但**不是合法 JSON**，且新旧版本写法不同：

| 事实 | 数值 / 说明 |
|---|---|
| 非标准：裸数字键 | 60 处，含负数（`{...,8376:{...},-103:{...}}`） |
| 非标准：非法转义 | 20 处 `\“` `\”`（中文引号被转义） |
| BOM | 无 |
| 顶层 | `{"playerStore":{"__type":...,"value":{...}},"storeStation":{...}}` |
| 字段规模 | 8,122 个节点（`playerStore` 144 个字段） |
| 新版存档差异 | 游戏新版本写成**带缩进 + CRLF** 的形式（`\r\n` + 空格缩进），解析器两种都要吃 |
| 旧副本 / 新版 | 1,720,561 B（单行）/ 1,827,432 B（缩进版） |

**库存是「JSON 里的 JSON」**（这是最要紧的部分）：

* 字段：`playerStore.mainInvJSON`（当前库存，解码后 1,192,336 字符）、`playerStore.soldInvJSON`
  （已售出，≈76 KB / 22 件）
* 内层结构：`{"saveItems":[ ...物品... ]}`，每件物品约 19 个字段：
  `identifier, unitCount, uuid, itemShape, itemModifiedShape, itemType, name, shortDescription,
  flavorText, uniqueId, unitBaseValue, unitValue, lateUnitValue, backupUnitValue, spriteAtlasPath,
  spritePath, itemTypes, childItems, childItemInventoryNode, _keys, _values`
* `childItems` 是 **uuid 数字数组**（引用同一数组里其它物品的 `uuid`），不是嵌套对象；有 `childItems`
  的就是容器（如 `storage_bay` / `smuggler_bay_mod` / `sec_box`），`itemTypes` 里带 `STORAGE`
* 物品名 `name` 是中文；`identifier` 是英文 id
* **`unitCount` 在这份存档里 287 件全是 1** —— 游戏没有堆叠概念，数量列意义有限
* 属性在 `_keys`（tag 名数组）与 `_values`（同长度对象数组）里，**两者必须一一对应**；
  `_values[i]` 的字段：`<identifier>k__BackingField`（tag 名）、`<identifierName>k__BackingField`、
  `valueEnabled`、`internalValueString`、`valueInt`、`valueFloat`、`valueLong`、`valueDouble`、`valueBool`

已知 tag 语义（用户确认 / 实测）：

* `CUSTOM_NAME_TAG` 的 `internalValueString` = 玩家给物品起的自定义名（参考存档里 5 件）
* `BONUS_PERCENTAGE_PERFORMANCE_INT` / `_EFFICIENCY_INT` / `_QUALITY_INT` = 模组性能/效率/质量加成，
  值在 **`valueInt`**（45 件物品有）；同名 `TEMP_PERCENTAGE_*` 是当前生效值，通常与 BONUS 相同
* 词典：`TAG_CN` / `TYPE_CN`（HTML 内），参考存档里出现的 236 种标签 + 33 种 itemTypes 已全覆盖
  （共 269 条）。**遇到未翻译的标签，只需往 `TAG_CN` 加一行**，UI 会自动用上；缺失时 UI 显示英文原名
  并标「未翻译」，这是有意设计，不要改成硬编码中文

## 3. 架构（单文件内的分层）

```
① 宽容解析器        parseLoose(text)   → 节点树，每个值记录 start/end 字符偏移
                    支持裸数字键、非法转义、尾逗号
                    annotate(root, segs) 给每个节点挂路径段（UI 显示时跳过 Unity 包装层 'value'）
                    lookup(root, 'a.b[2].c') / lookupBySegs(root, segs) 按路径取节点
② 导出              buildOutput(text, changes)  按偏移**从后往前**替换；change 可带 literal
                    patchLiteral(literal, edits) 转义感知局部改写（内嵌 JSON 字符串用）
                    encStr / encVal 生成标准转义；verifyOutput 导出后按路径回读自检
③ 内嵌 JSON 子结构  isJsonStr / ensureSub(node)  懒解析成 node.sub {text, root, changes, literal}
                    subsOf(root) / buildSub(sub) 递归写回（支持 JSON 里再套 JSON）
④ 状态              S = {text, root, changes, subs, subByLabel, sel, rowByNode, ...}
                    每个节点用 node._map 决定改动写进哪张 Map（S.changes 或 sub.changes）
⑤ UI                常用数值面板 / 库存页 / 全部字段树 + 搜索 / 改动清单
                    库存页：invRows → renderInv（INV_COLS 统一列宽）→ 行 emitRow
                    tagBlock(row) = _values 属性编辑器；tagValueField 按 tag 后缀挑可编辑字段
⑥ 持久化            IndexedDB（库 es3-save-mod / key lastSave）缓存原文 + 改动路径(segs)+值
                    restoreLast() 打开页面自动恢复；「忘记上次」清除
```

## 4. 硬约束（改了会出事）

1. **导出只能是「按字符区间局部替换」**。禁止把整份存档 parse 成对象再 `JSON.stringify` 重新序列化
   —— 会丢掉原文的裸数字键、`\/` 等转义写法与缩进，并且改动范围不可控。
   验收标准：**不改任何值时，导出的文件与输入逐字节相同**。
2. **不要改动 `_keys` / `_values` 的结构**（不新增、不删除项）。二者长度必须始终相等，游戏中按索引
   对应读取。当前版本不支持「给物品添加一个新 tag」「新增物品条目」这类操作。
3. **表头与数据行必须共用同一份列宽定义**（`INV_COLS`）。行内的按钮/输入框宽度都从 `INV_COLS[i].w` 取，
   不要写死数值 —— 历史上因为表头写 52px、行里写 60px 出现过 8px 列错位。
4. **CSS 里的 `input{min-width:0}` 不能删**。flex 行里 `input` 的 `min-width:auto` 会按内容撑到约
   160px，直接毁掉表格列对齐。
5. **所有界面内容必须由存档数据驱动**，不要硬编码物品清单 / 类型清单 —— 玩家尚未获得的物品、未来版本
   新增的物品与标签都要能打开和编辑（词典缺失时回退英文即可）。
6. 改动写回时，节点必须挂上正确的 `_map`（库存内层节点 → 该内嵌 JSON 的 `changes`），否则同步/撤销/
   导出会漏改或写错层级。

## 5. UI 约定（用户明确要求过的）

* 分组后**默认全部收起**，提供「全部收起 / 全部展开」；切换分组方式时重新收起
* 批量修改用**选中项 + 同步**模型：行首复选框、组头「选中本组」、表头全选；
  「批量同步」开关打开时，改选中项里任意一件的属性会写到其余选中项（类型不一致的跳过并提示）
* 模组不设独立的「加成」按钮 —— `_values` 属性编辑器已经覆盖
* 主题：跟随系统深浅色；界面中文；数字列右对齐；行有斑马纹
* 库存页首屏简洁：靠搜索/分组/折叠按需展开，但**保留手动改任何值的能力**

## 6. 测试方法（可复现）

测试脚本放在仓库外（如 `%LOCALAPPDATA%\Temp\es3test`），不提交。三种手段：

**a. 抽 core 逻辑跑 Node**（不需要浏览器）：从 HTML 里截取 `<script>` 开头到 `const S = {` 之前，
用 `new Function(core + 'return {parseLoose, annotate, lookup, lookupBySegs, buildOutput, patchLiteral, verifyOutput, encVal, encStr, friendlyPath};')()` 拿到 API。

**b. jsdom 测 UI**：`new JSDOM(html, {runScripts:'dangerously', virtualConsole})`，然后 `window.load(text, name, false, bytes)`；
用 `w.eval("...")` 访问内部函数（顶层 `function` 声明会挂到 window，`const S` 需要 `w.eval` 才可见）。
拦截下载：把 `window.Blob` 换成捕获构造函数、`URL.createObjectURL` 打桩、`HTMLAnchorElement.prototype.click` 置空。

**c. 真实浏览器测布局**（jsdom 不做布局计算，列对齐必须这样测）：
```js
const puppeteer = require('puppeteer-core');
puppeteer.launch({ executablePath: 'C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe', headless: 'new' })
```
用 `(await page.$('#file')).uploadFile(存档路径)` 载入存档，再用
`getBoundingClientRect()` 比较表头与数据行每一列的 `left` 是否完全一致。

必测清单：宽容解析统计 → 无改动导出逐字节等于原文 → 库存值改动后回读正确、其余物品不变 →
搜索能进内嵌 JSON → 分组收起/展开 → 选中 + 同步 → 刷新页面自动恢复 → 10 列几何对齐 → 控制台 0 错误。

## 7. 待办 / 已知限制

* `TEMP_PERCENTAGE_*`（当前生效加成）暂未单独提供编辑入口，改 `BONUS_*` 时不会同步改它 —— 待确认游戏
  实际按哪个计算
* 不支持新增物品条目 / 新增 tag（见硬约束 2）；若要做，必须同时维护 `_keys` 与 `_values` 的一致性
* 未翻译标签靠人工补充 `TAG_CN`
* 页面是单文件（用户明确允许拆分，但当前保持单文件便于复制/备份）

## 8. 版本

`0.1.0` —— 首个打 tag 的版本。
