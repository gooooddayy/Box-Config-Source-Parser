# 开发全史（DEVLOG）

> 项目从 2026-08-23 创建到 2026-10-09 的完整演进记录。
> 按时间线组织，每段只留"做了什么 + 为什么"。踩坑细节见 `PITFALLS.md`。

---

## 阶段一：打地基（2026-08-23 ~ 08-25）

### 08-23 项目创建
- 创建 `tvbox-parser.html`：单文件零依赖工具。
- 功能：文本/URL/文件输入 → 提取链接并 fetch → JSON 解析 → 分类展示
  （sites / lives / parses / wallpapers / rules）→ 勾选/全选/反选 → 复制/导出 data.json。
- 关键实现：URL 正则提取、嵌套 URL 递归解析、CORS 兜底、去重、XSS 转义。
- 预修：fetch 超时+重试、非 JSON 文本兜底、BOM 清理、大数据 max-height 截断。
- **8-24 重大修复**：导出时丢弃了 TVBox 顶层全局字段（spider/logo/wallpaper/danmaku/ijk/flags…），
  没了 spider 搜索加载全失效 → 新增 `extractGlobalFields()` 提取 `GLOBAL_KEYS` 顶层字段。

### 08-24 输入区改造 + 解析成功率升级
- 移除「示例」按钮；输入改为**显式 mode 驱动**：
  - URL tab → `mode 'url'`：从文本正则提取链接 → fetch。
  - 粘贴文本 tab → `mode 'json'`：直接 `JSON.parse` 整段（不再提取内部 URL 去误请求）。
  - 上传文件 → `mode 'auto'`：先试 JSON，失败再提取链接；支持多选。
- `parseAll` 重写为 mode 感知的工作单元列表（`handleDirect` / `handleUrl`）。
- URL 提取增强：终止符含全角标点，**不含 `.` 与 `?`**（主机/查询串必需）；自动去重 + 长度 ≥10。
- JSON 解析三级容错：严格 → 去尾随逗号 → 剥离外层包裹提取首对 `{}`/`[]`。
- **网络层改并发竞跑**（原 5 个代理串行 75s → 直连+代理同时发，谁先成功算谁）。
- 并发解析 `runPool(limit=4)`。

### 08-25 崩溃修复 + 两个致命配置问题定位
- **92 条链接崩溃**：`renderLog` 全量重绘 O(n²) + `handleUrl` 无限制跟随子链接
  → rAF 合并 + 日志上限 300 + 子链接上限 3 条 + 响应 >800KB 跳过子链接。
- **「配置解析失败」根因**：`parses[].ext` 是字符串 → 见 PITFALLS A1。
  新增 `coerceExt()` 把 JSON 字符串还原成对象。
- **「网络错误 / jar load err」根因**：顶层 spider 是相对路径 `./OS.jar`。
  新增 `currentBaseUrl` 记录来源地址 + `resolveSpiderPath()` 拼绝对。
  兼容模式安全网：拼不活就 `delete output.spider`。

---

## 阶段二：与"死链"缠斗（2026-08-26 ~ 08-28）

### 08-26 spider 策略四连改（每次都推翻上一次）
- **v1 写死覆盖**：把 spider 写死成 <某jar> → 用户质疑"不同源各有自家 spider，写死不对"。
- **v2 跟随源 + 黑名单**：只过滤内网穿透/localhost → 翻车（用户源的 spider 全在
  `<某失效域>` punycode 域名和 `<某失效域>` 上，全是 404 但不在黑名单）。
- **v3 反向判定（只拦可疑）**：相对路径 / 内网 IP / localhost / punycode / 已知失效域
  → 又被裸 IP `http://<裸IP>/OS.jar` 绕过。
- **v4 正向白名单**：`TRUSTED_SPIDER_HOSTS` 只收录长期稳定基础设施（GitHub raw / jsDelivr /
  知名 ghproxy 镜像 / <某托管域名> / <某托管域名>）；多源一律回退默认。**彻底解决问题。**
- 站点级 jar 同步净化（可疑域 → 重写为全局 spider）。
- **教训**：多作者源合并时，源 spider 主机是任意公网 IP / 临时域名，黑名单永远补不全；
  浏览器端又无法做实时可达性检测（CORS）。⇒ 只能正向白名单。
- **⚠️ 教训修正**：任何"默认兜底 jar"都会随时间失效（<某jar> 后来就不可达了）。
  真正的长期方案不是"找个永不失效的默认"，而是**给出可更换的默认 + 清晰的替换指引**。

### 08-26 v5 自动探测可用 spider（用户零操作）
- 候选池 `getSpiderCandidates()`：内置候选 + **从用户实际解析的源中学习**
  （`rememberSpiderCandidate`）+ localStorage 缓存。
- 页面打开时恢复上次可用 spider，后台自动探测（`autoProbeSpiders()`）：
  并发竞跑代理，判据 = 200 / 非 HTML / ≥50KB / 文件头是 ZIP 或 DEX。
- 探测结果写入 localStorage + 输入框默认值。

### 08-27 三个真实源的"搜索为空"（认知连续推翻）
- **源 A（<某托管域名> 生态）**：站点全是 `csp_XxxGuard`，真实 spider 是
  `http://<某对象存储>/.../xxx.jpg;md5;<md5已隐去> + .jpg 伪装）。旧解析器要求 HTTPS+白名单
  → 被判不可信 → 回退成不含 Guard 类的 <某jar> → 搜索为空。
  ⇒ 认知：**TVBox 加载 jar 不要求 HTTPS、不看扩展名。**
- **源 B（<某源域名>）**：spider 是相对路径 `./lib/<某jar>` → 被判不安全 → 回退错 jar。
  新增 `inferBaseUrl()`（从 spider/站点 api 推断来源 base，无 URL 来源时也能拼绝对）。
- **源 C（<内网穿透域名>）**：spider 是 `http://<某IP:端口>/vip/jar/<某jar端点>`
  （裸 IP + HTTP + .php）→ 被"裸 IP 不可信"规则回退 → 又是错 jar。
  ⇒ **发现 UA 校验**：浏览器 UA 返回 HTML 广告页，`okhttp/3.15` 才返回真 jar。
  ⇒ 认知：**不能凭浏览器探测结果决定是否跟随源 spider。**
- **最终原则确立**：单源 = 无条件跟随源 spider（只要是 http/https 绝对地址）；
  多源 = 默认兜底 + 保留 per-site jar。

### 08-28 多源自动选 spider + 多仓功能
- `dependsOnSpider(site)`：type 3/4 且 api 是类名（无 scheme/斜杠/点）且无 per-site jar → 依赖。
- `chooseBestSpider()`：统计各源 spider 覆盖的"依赖站点数"，选覆盖最多者
  （评分 = dependent×1000 + total；上次实测可用者 +50万 加权）。
- 新增「多仓」按钮（与「解析」并列）：输入 URL 列表 → 生成 TVBox 多仓 json。
- 多仓输入格式增强：支持「名字，链接」（中英文逗号），导出名 `data.json`。

---

## 阶段三：架构级改造（2026-08-30 ~ 09-08）

### 08-30 三件大事

**① 解析速度优化（修自己引入的回归）**
- 疏漏：实测确认可用的 `proxy.cors.sh` **漏写进代码**，池里全是死通道。
  ⇒ 教训：实测出的可用通道必须当场核对是否真的写进了代码。
- 回归：四级串行编排最坏 31s → 改 `stagedRace` 分级竞跑，最坏 ~10s。
- `FETCH_POOL` 6→2 是为缓解 429，但限流的是特定代理 → 恢复为 4。
- **新增 `toCdnUrl()`**：GitHub 系链接自动转 jsDelivr（返回 `ACAO:*`，浏览器可直连，
  完全绕过 CORS 代理）。必须优先匹配 `refs/heads/分支`，否则 `refs` 被误当分支名 404。

**② 多配置源失效排查：不是多仓冲突**
- 结论：代理全灭（环境）+ 并发过高（429）+ 容错太弱，三个独立问题叠加。
- **新增 `sanitizeJsonText()`**：字符串感知单趟扫描，同时处理 `//`、`/* */` 注释与
  字符串内未转义的控制字符。**只净化字符串内内容，绝不能动结构层**（否则 `http://` 被拦腰截断）。

**③ ★ per-site jar 注入（项目最重要的一次修复）**
- 问题：多作者多源合并后大量站点搜索无结果。
- 量化：1038 个依赖站点需要 551 种类，单 jar 最多覆盖 8%。
- 方案：`handleDirect` 合并时给每个站点打 `__srcSpider` 标记 →
  `applyPerSiteJar()` 给"无 jar 且依赖 spider"的站点注入其所属源的 spider。
- 效果：3 源合并场景覆盖率 63.6% → **99.5%**。
- 代价：首次加载需下载约 33 个 jar（~50MB），之后有缓存。**值得**。

### 08-30 多仓（订阅）格式修正
- 根因：裸数组 `[{name,url}]` → TVBox 只认 `{urls:[...],vip:[...]}` → "订阅有问题"。
- 修复 `multiRepoJSON()` + `parseMultiRepoJSONBlob()`（输入侧兼容读取，保留仓库名）。

### 09-08 41 站全灭救援 + 体检功能

**① 41 站全部失效的三层根因**
1. 全局 spider 404（主因，导致全军覆没）。
2. jar-D（`<某托管域名>/jar/<某jar>`）失效 → 8 站死亡。
3. 2 站无 jar，只能依赖已 404 的全局 spider。
- 修复：全局 spider 换成含豆瓣类的 HTTPS jar；per-site jar 全 HTTP→HTTPS；
  给无 jar 站注入可用 jar；**删除 8 个无法修复的站**（三个 jar 全都没有对应类）。

**② 用户要求"尽量救回" → 全部救回**
- <某托管域名> 全域名变体（42 组合）全返回 404 页 → 桶确实清空。
- **GitHub code search 反查** → 找到 <某公开镜像仓库> 的 <某公开配置>，
  其 `<某jar>` 的 md5 与失效文件**完全一致** → jsDelivr 镜像拉取成功 → 8 类全中。
- ⇒ **确立方法论：md5 反查镜像仓库**（见 PITFALLS D3）。

**③ 新功能：「🩺 体检优化」**（后被移除）
- 一键：下载 jar → 校验 PK 头/md5 → 解析 dex 类 → 与 csp_Xxx 比对 → 从可用 jar 池自动救回。
- 零依赖实现：纯 JS md5、最小 zip 读取（EOCD → 中央目录 → `DecompressionStream('deflate-raw')`）、
  dex uleb128 解析、二进制竞跑。

---

## 阶段四：合规化 + 体验打磨（2026-09-30 ~ 10-09）

### 09-30 内置豆瓣置顶
- `BUILTIN_DOUBAN_SITE` 常量 + `buildExportJSON` 无条件置顶（先剔除用户源里的豆瓣站防重复）。
- **⚠️ 后来全部撤销**（合规原因），见 10-09。

### 10-09 大改造（一天内 6 轮）

**① 配置源诊断**
- 线上 data.json 诊断：HTTP 200、JSON 合法、与桌面版逐字段一致 → **文件本身没问题**。
- 网络：github.io 直连通、jsDelivr 间歇、<某对象存储> jar 直连通。

**② 合规化改造（把工具变成"中立工具"）**
- 删除 `BUILTIN_DOUBAN_SITE` → 改为「豆瓣置顶配置」粘贴框存 localStorage。
- 清空 `spiderOverrideUrl` 默认值、`FALLBACK_SPIDER` 清空为 `''`。
- 删除 `getSpiderCandidates()` 里的 7 个硬编码候选 jar（只剩本机历史 + 用户源自带）。
- 保留 `TRUSTED_SPIDER_HOSTS` 裸域名白名单（裸域名 ≠ 资源指针，功能必需）。

**③ 二次合规化 + 命名中性化**
- **删除「体检优化」整段**（用户要求）：UI + 约 256 行 JS，0 残留。
- 命名中性化：标题「配置解析器」、标签「链接 / 粘贴 JSON / 上传文件」、
  结果「站点 / 直播源 / 解析 / 壁纸 / 规则」、页脚改一行「纯本地运行 · 数据不会上传到任何服务器」。
- 删掉免责声明（用户："不必要声明也不要"）。
- 上手体验：**输入智能识别**（`looksLikeJsonText`，链接框粘 JSON 也认）、
  导出卡重构（可见区只留「兼容模式」「豆瓣置顶」+ 高级选项折叠）、Ctrl/Cmd+Enter 快捷解析。
- 修复真实逻辑缺口：`chkDoubanPin`/`chkSpiderOverride` 补 `change` 监听。

**④ 结果内名称搜索（类 Ctrl+F）**
- `#resSearchBar` 一行紧凑搜索条：输入即高亮（防抖 180ms）、Enter/▼ 下一个、
  Shift+Enter/▲ 上一个、**到尾部循环回卷**、Esc 清空、`2/5` 计数。
- 作用域 = **仅当前标签页的名称列**；切 tab 自动重新定位；重新解析自动清空。

**⑤ 豆瓣置顶改默认开启 + 取消"解析后默认全选"**
- 用户要求：豆瓣置顶默认勾选；解析结果**全不选**，用户手动勾。

**⑥ 多段拼接 JSON 合并（解决"只有前面几条站点能用"）**
- 用户桌面 data.json 是 **4 个独立 JSON 对象首尾拼接**（4 次导出被拼一起）。
- 47 站全是 type:3 的 csp_ 站点，分属 **4 个不同 jar** → 必须靠站点级 jar 才能共存。
- 新增 `splitJsonSegments()` + `parseAll` 统一入口 + `dedupSites` key 唯一化 +
  `exportNodes` 放行字符串条目（修好壁纸恒为 0 的 bug）。
- 产出 41 站合并版。

**⑦ 死 jar 换同 md5 镜像（"只有 25 个有效"的根因）**
- 段 4 的 `<某域名:端口>/spider.jar` **整站 404**，12 站全灭。
- md5 反查 → <某公开镜像仓库> 镜像（ghfast.top / gh-proxy.com 双通）→ md5 完全一致 → 41/41 类覆盖。
- 顺带修复 2 个站的 ext 令牌（被前置了失效主机，改回裸令牌）。

**⑧ 解析器 img+ 支持 + jar 依赖清单**
- `img+`（图片伪装 jar）全链路支持（此前会被误判成相对路径，甚至覆盖掉站点级 jar）。
- 导出审核新增 `summarizeJarDeps()` —— 输出"本次导出共依赖 N 个 jar：host（全局，X 站）…"。
- 删掉 2 条与事实矛盾的提示（"属兼容限制，不影响配置加载"）。
- L1Box 源码级确认：站点级 jar 完全支持（独立 DexClassLoader），加固 jar 有整包跳过保护。

---

## 关键决策速查

| 决策 | 时间 | 理由 |
|---|---|---|
| 正向白名单选 spider（不做黑名单） | 08-26 | 多作者源的失效域永远枚举不完 |
| 单源跟随源 spider / 多源用兜底 | 08-27 | 单源最稳；多源全局槽位只有 1 个 |
| per-site jar 注入是核心机制 | 08-30 | TVBox 架构限制的唯一破局点 |
| 死 jar 用 md5 反查镜像救援 | 09-08 | 换托管位置 md5 不变 |
| 工具彻底中立化（0 资源指针） | 10-09 | 公开托管不担侵权风险 |
| 结果默认不全选 | 10-09 | 用户明确要求手动勾选 |
| 豆瓣置顶默认开（只识别用户源里已有的） | 10-09 | 工具不内置数据也能满足诉求 |
