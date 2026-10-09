# 代码架构（ARCHITECTURE）

> `tvbox-parser.html` 单文件三段的内部结构、关键函数职责、数据流。
> 后期改 bug / 加功能前先看这份，能快速定位该动哪里。
>
> ⚠️ **本文所有行号基于 2026-10-09 版本**（2312 行 / 120KB）。改代码后行号会漂移，
> 用函数名搜索定位，别死记行号。

---

## 一、文件整体结构

单个 HTML 文件，零依赖、零外链（除运行时 fetch 目标外）：

| 段 | 行号范围 | 说明 |
|---|---|---|
| `<style>` | 7 – 131 | 全部样式（约 125 行，紧凑） |
| `<body>` | 133 – 270 | 5 张卡片 + 页脚 |
| `<script>` | 272 – 2307 | 全部逻辑（约 2035 行，含注释与空行） |

**⚠️ 注意**：文件里大量 `data-page-node-id="xxxx"` 属性是页面编辑器生成的，**与逻辑无关**，
清理时可忽略或保留（删掉不影响功能，但重排可能影响用户已保存的页面）。

---

## 二、页面区块（body）

| 卡片 | id | 作用 |
|---|---|---|
| 输入卡 | （无 id，第一个 card） | 三个 tab（链接 / 粘贴 JSON / 上传文件）+ 解析按钮 + 多仓按钮 |
| 日志卡 | `#logCard` | 解析过程日志（默认隐藏） |
| 结果卡 | `#resultsCard` | 统计 + 5 个分类标签 + **名称搜索条** `#resSearchBar` + 数据表 |
| 导出卡 | `#exportCard` | 兼容模式 / 豆瓣置顶 + 高级选项折叠 + 审核提示 + 预览 + 导出按钮 |
| 多仓卡 | `#multiRepoCard` | 多仓 JSON 预览 / 导出 / 改名 |

**结果卡的搜索条**（`#resSearchBar`，第 200–205 行）：
`#resSearchInput` 输入 → `#resSearchCount` 显示 `2/5` → `#btnSearchPrev`/`#btnSearchNext` 前后跳。

---

## 三、JS 模块地图（按 `// =====` 分块）

| 行号 | 模块 | 关键函数 |
|---|---|---|
| 276 | State | 全局状态（`selections` / `allNodes` / `sourceSpiders` 等） |
| 291 | DOM refs | `$` / `$$` |
| 295 | Toast | `toast(msg)` |
| 304 | Tab switching | 统一 tab 事件委托（**新增 tab 只需加 data 属性**） |
| 330 | Input gathering | `looksLikeJsonText` / `gatherInputs` / `readFiles` |
| 362 | URL Extraction | `URL_RE` / `extractUrls` |
| 387 | Fetch（CORS 竞跑） | `toCDNUrl` / `stagedRace` / `fetchUrl` / `looksLikeConfig` / `unwrapProxy` |
| 557 | Parse Content | `safeParse` / `sanitizeJsonText` / `tryParseJSON` / **`splitJsonSegments`** |
| 659 | 归一化 | `normalizeSite` / `normalizeLive` / `normalizeParse` / `extractNodes` |
| 747 | 全局字段 | `GLOBAL_KEYS` / `resolveSpiderPath` |
| 767 | spider 自动探测 | `getSpiderCandidates` / `autoProbeSpiders` / `rememberSpiderCandidate` |
| 899 | spider 判定 | `TRUSTED_SPIDER_HOSTS` / `isSpiderTrusted` / **`isSpiderSafe`** / `dependsOnSpider` / `chooseBestSpider` |
| 999 | 来源推断 | `inferBaseUrl` / `extractGlobalFields` / `flattenTVBox` |
| 1073 | 去重 | `dedupSites`（含 key 唯一化）/ `dedupSimple` |
| 1107 | 并发池 | `runPool` |
| 1119 | **主解析** | `parseAll` / `handleDirect` / `handleUrl` / `mergeNodes` |
| 1300 | 进度/日志 | `updateProgress` / `renderLog`（均 rAF 合并） |
| 1353 | 结果渲染 | `CAT_CONFIG` / `renderResults` / `renderSection` / `esc` |
| 1466 | **结果内搜索** | `sPanel` / `sRebuild` / `sStep` / `sFocus` |
| 1522 | 选择事件 | `refreshSelectionUI` / `updateRowCheckbox` |
| 1573 | 工具栏 | 全选/反选/清空 + `updateExportInfo` |
| 1620 | **导出** | `coerceExt` / `exportNodes` / **`applyPerSiteJar`** / **`buildExportJSON`** / `summarizeJarDeps` |
| 1871 | 兼容审核 | `compatAudit` |
| 1971 | 预览渲染 | `renderGlobalPreview` / `renderExport` / `renderAudit` |
| 2082 | 文件清单 | `renderFileList` |

---

## 四、关键数据流

```
输入（三种）
  ├─ URL 文本  → extractUrls → fetchUrl(CORS竞跑) → 得到配置文本
  ├─ 粘贴 JSON → 直接作为文本
  └─ 上传文件  → 先试 JSON，失败再提取链接
        ↓
   文本 → sanitizeJsonText（净化注释/控制字符）
        ↓
   tryParseJSON ─ 失败 → splitJsonSegments 切多段（每段一个 direct work）
        ↓
   parseAll 遍历 works：
     handleDirect(源对象)  ← 记录 currentBaseUrl、sourceSpiders、给站点打 __srcSpider
     handleUrl(url)        ← fetch → 递归
        ↓
   flattenTVBox → extractNodes → normalizeSite/Live/Parse（相对路径→绝对、类型推断、ext 还原）
        ↓
   dedupSites（key 唯一化）→ 写入 allNodes
        ↓
   renderResults（分类表 + 搜索条）
        ↓
   用户在导出卡勾选 → buildExportJSON：
     1. 组装 5 类节点（exportNodes：coerceExt / 补默认值 / 过滤非对象）
     2. mergeNodes → applyPerSiteJar（注入站点级 jar）
     3. 豆瓣置顶（chkDoubanPin，默认开）
     4. 全局字段 + spider 决议（单源跟随 / 多源 chooseBestSpider / 兜底）
     5. compatAudit 审核 → renderAudit 输出提示 + summarizeJarDeps 清单
        ↓
   复制 / 导出 data.json
```

---

## 五、核心函数详解（改代码必读）

### `parseAll()` — 主入口
- 重置 `selections`（**空 Set**，即解析后默认不全选）、`allNodes`、`sourceSpiders`。
- 收集 works：`{kind:'direct'|'url', name, obj|url}`。
- direct 走 `handleDirect`，url 走 `handleUrl`（并发池）。
- 结束时 `renderResults()` + toast。

### `handleDirect(name, obj)` — 单份配置入口
- `inferBaseUrl(obj)` → 设 `currentBaseUrl`（贴 JSON 无来源时也能拼相对路径）。
- `extractGlobalFields` → `flattenTVBox` → `extractNodes` → 归一化。
- 给每个站点的 `raw` 打 `__srcSpider`（该站所属源的 spider），供后续 `applyPerSiteJar` 用。
- 累加 `sourceSpiders`（源 spider → 覆盖的依赖站点数）。

### `applyPerSiteJar(sites, resolvedSpider)` — 站点级 jar 注入 ★
- 自带绝对 jar → 不动（尊重源配置）。
- 相对 jar → 解析为绝对，失败则用源 spider。
- 无 jar 且 `dependsOnSpider()` → **注入该站点的 `__srcSpider`**（与全局 spider 相同则跳过，避免冗余）。
- 结束 `delete s.__srcSpider`（内部标记绝不写入导出文件）。

### `buildExportJSON()` — 导出组装
顺序很重要，改这里要完整读一遍：
1. 站点/直播/解析/壁纸/规则（各自严格过滤开关）→ `exportNodes`。
2. `mergeNodes` → `applyPerSiteJar`。
3. **豆瓣置顶**（`chkDoubanPin` 勾选时：在所选站点中找 `api` 含 `NewDouBan` 或 `name` 含「豆瓣」
   的站，`di > 0` 才 splice+unshift）。**工具本身不含任何豆瓣数据。**
4. 全局字段合并（`chkGlobal`）+ spider 决议（`chkSpiderOverride` 强制 / 单源跟随 / 多源选优）。
5. 兼容模式（`chkCompat`）：spider 不可用则剥离。
6. `delete` 所有 `__` 前缀内部字段。

### `compatAudit(obj)` — 导出前预检
返回 `{passed, warnings:[{lv:'error'|'warn'|'info', msg}]}`。
检查项：站点 key/api 完整性、parses.ext 类型、spider 可用性、jar 依赖清单（`summarizeJarDeps`）、
多源合并提示。

### 结果内搜索 `sRebuild` / `sStep` / `sFocus`
- `sRebuild()`：按 `#resSearchInput` 的值在**当前 tab 面板**的名称列（`.col-name`，
  壁纸/规则无名称列则回落 `.col-api`）里找命中，写 `tr.hit` / `tr.hit-cur` class。
- `sStep(d)`：±1 移动当前命中，**到边界回卷**（`(sCur + d + len) % len`）。
- `sFocus()`：`scrollIntoView({block:'center', behavior:'smooth'})` + 更新计数。
- `renderResults()` 开头会清空搜索；tab 切换时 `sRebuild()` 在新列表重新定位。
- **CSS 注意**：`tr.hit td` / `tr.hit-cur td` 必须定义在 `tr.selected td` **之后**。

---

## 六、关键常量（改行为先看这里）

| 常量 | 位置 | 值 / 含义 |
|---|---|---|
| `URL_RE` | L368 | URL 提取正则（终止符含全角标点，不含 `.`/`?`） |
| `FAST_TIMEOUT` / `SLOW_TIMEOUT` | L400/401 | 5s / 9s |
| `CDN_HOSTS` | L413 | jsDelivr 两个域名（`toCdnUrl` 用） |
| `PROXY_FAST` / `PROXY_SLOW` | L435/443 | 两波 CORS 代理池 |
| `GLOBAL_KEYS` | L747 | 需要从顶层提取的全局字段清单 |
| `FALLBACK_SPIDER` | L765 | **空字符串**（合规化后不再硬编码） |
| `TRUSTED_SPIDER_HOSTS` | L899 | 裸域名白名单（功能性，非资源指针） |
| `MAX_SUB` / `MAX_RESP` | L1252/1253 | 子链接上限 3 / 响应上限 800KB |
| `MAX_LOG` | L1320 | 日志上限 300 条 |
| `CAT_CONFIG` | L1354 | 五个分类的表头/字段/默认值配置 |
| `NODE_DEFAULTS` | L1634 | 各类节点的字段默认值 |

---

## 七、修改指南（高频改动点）

| 想改什么 | 动哪里 | 注意 |
|---|---|---|
| 新增一个分类（如 `ads`） | `CAT_CONFIG` + `ALL_CATS` + 结果区 tab + `GLOBAL_KEYS`/`NODE_DEFAULTS` | 同步 4 处，漏一处会出 undefined |
| 调整导出字段默认值 | `NODE_DEFAULTS` | 只在字段缺失时生效 |
| 新增导出开关 | 导出卡 HTML + `renderExport()` + **接 `change` 监听** | 不接监听预览不刷新（踩过） |
| 改 spider 选择策略 | `isSpiderSafe` / `isSpiderTrusted` / `chooseBestSpider` | 三个函数语义不同，别混 |
| 改搜索行为 | `sRebuild` / `sStep` + `.res-search` 样式 | 高亮 CSS 顺序 |
| 改审核提示 | `compatAudit` / `summarizeJarDeps` | 提示要与实际行为一致（删过矛盾文案） |
| 加输入方式 | `gatherInputs` + tab HTML | 走统一 works 入口，别另开路径 |

---

## 八、验证方式（改完必跑）

1. **语法**：抽出 `<script>` 内容跑 `node --check`。
2. **功能**：jsdom 真实页面级（能跑完整 解析→勾选→导出 链路），见 `PITFALLS.md` C4/C5。
3. **视觉**：Chrome 无头截图目视（见 C3）。
4. **合规**（若对外发布）：扫描 HTML 内所有 `http(s)://`，确认只有裸域名（CORS 代理 / CDN）与占位符，
   无任何具体资源指针（jar 路径、`;md5;` 真实值）。
