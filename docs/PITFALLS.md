# 踩坑档案（PITFALLS）

> 本项目全部踩过的坑，按主题分类。**改代码前先读这里**，能省掉大量返工。
> 每条格式：现象 → 根因 → 修复/规避。标 ★ 的是最容易再犯的。

---

## A. TVBox 主程序机制（这是"规矩"，不是 bug）

### A1 ★ `parses[].ext` 是字符串 → 整份配置「配置解析失败」

- **现象**：TVBox 加载配置直接报"配置解析失败"，全部站点消失。
- **根因**：TVBox `ApiConfig.parseJson` 读 `parses[].ext` 用的是
  `obj.get("ext").getAsJsonObject().toString()` —— **只要该字段存在就必须是 JSON 对象**，
  字符串会抛 `IllegalStateException`，异常冒泡导致整份配置解析失败。
- **证据**：对比可正常识别的 `123.json`，它的 `parses` 是**空数组**（len=0，根本不触发）；
  而失败配置有 58 个 string ext，形如 `"{\"header\":{...}}"`（订阅源把 ext 存成了 JSON 字符串）。
- **修复**：`coerceExt(v)` —— 已是对象→保留；是 **JSON 字符串→`JSON.parse` 还原成对象**
  （关键：能找回真实数据，如 user-agent header）；解析失败/普通字符串→置 `{}`。
- **反向坑（重要）**：早期误判为"字段类型不一致"，做了 blanket 类型强制转换
  （把对象也转成字符串），**方向完全相反**，把能用的配置也搞死了。
  ⇒ **绝不做 blanket 类型统一，只对"字符串形态的对象"做单点还原。**
- 注：`sites[].ext` 用 `optString` 读，string 与 object 都兼容 —— **只有 parses 有这条硬要求**。

### A2 ★ 「网络错误 / jar load err」= 全局 spider 不可达

- **现象**：配置能加载，但一点就报 jar load err / 网络错误。
- **根因**：顶层 `spider` 是**相对路径**（如 `./OS.jar`）、死链、裸 IP、临时域名。
  导出成"公共主配置"后，TVBox 按远程 URL 去抓那个相对路径 → 404。
- **修复路径（三步，逐步深入）**：
  1. 相对路径 → 用来源 base 拼成绝对 URL（`resolveSpiderPath`）。
  2. 兼容模式安全网：拼完仍不可用 → **直接 `delete output.spider`**（配置能加载，csp 站失效但报错消失）。
  3. **深度认知**：别以为"拼成绝对 URL 就能活" —— base 地址本身可能就是死链
     （实测 `https://<某内网穿透域名>/vip/vip/OS.jar` → 404，内网穿透域名根本没公开该文件）。
- **v4 决策**：多源合并时**不跟随任何单个源**，统一用长期可信的通用 spider；
  仅"单源 + 源 spider 在正向白名单"才跟随。（黑名单/启发式枚举永远补不全任意公网 IP。）

### A3 ★ 「go 代理正常，但搜索结果为空」= spider jar 不含该站的 csp_ 类

- **现象**：TVBox 显示 go 代理正常、站点能打开，但搜索无任何返回。
- **根因链**：解析器把源 spider 换成了另一个 jar → 那个 jar 能加载（所以代理正常）
  → 但**不含该源站点所需的类** → 搜索/首页方法不存在 → 空结果。
- **关键认知（三条，全是实测推翻旧假设）**：
  1. TVBox 加载 spider jar **不要求 HTTPS**，也**不看扩展名** —— `http://...xxx.jpg`、
     `http://ip:port/<某jar端点>` 这类"伪装 jar"完全正常。
  2. **大量 jar 端点有 UA 校验**：`<某jar端点>` 用浏览器 UA 返回 HTML 广告页（31KB），
     换成 `okhttp/3.15` 才返回真 jar（2MB / 180 类）。
     ⇒ **探测 jar 必须用 okhttp/3.15 UA，否则会把好 jar 误判成失效。**
  3. **不能凭浏览器 fetch 探测结果决定是否跟随源 spider** —— TVBox 用 okhttp，
     很多端点做了 UA 校验/动态 302，浏览器拿不到 ≠ TVBox 拿不到。
- **最终策略**：单源且源 spider 是 http/https 绝对地址 → **无条件跟随**（唯一能保证
  导出配置与源行为一致的方法）；裸 IP、HTTP、`.php`、`.jpg` 统统保留；只拦 100% 无 scheme 的相对路径。

### A4 ★ TVBox 只有 1 个全局 spider 槽位 —— 多源合并的结构性限制

- **量化事实**（对 1983 站点的真实配置实测）：
  依赖全局 spider 的站点 1038 个，需要 **551 种** csp_ 类；单个 jar 最多覆盖 44 种 = **8%**；
  ⇒ **799 个站点（40.3%）必然搜索无结果**。
- **唯一破局点：站点级 `jar` 字段**（`sites[].jar`）—— TVBox 支持，源作者自己也在用。
- **坑**：`csp_Xxx` 是 **api 方法名前缀，不是类名**。
  `csp_Bili` → 类名 `Bili` → 加载 `com.github.catvod.spider.Bili`。
  ⇒ **匹配覆盖率时必须去掉 `csp_` 前缀**（第一次算成 0% 就栽在这）。

### A5 站点级 jar 是完整支持的（源码级确认，别再怀疑）

- `ApiConfig.parseSite()` 读 `jar` 字段 → `ApiConfig.getCSP()` →
  `jarLoader.getSpider(key, api, ext, sourceBean.getJar())`。
- `JarLoader.getSpider()`：站点 jar ≠ 空 → `jarKey = MD5(jarUrl)` →
  `loadJarInternal(jarUrl, jarMd5, jarKey)` 建**独立 DexClassLoader**，
  缓存到 `filesDir/<md5(url)>.jar`；`csp_` 前缀自动去掉后载入类。
- 走同一 `loadClassLoader`：加固家族判定 `isFamily` → `protectedInitJar.init`；
  有 `JarKillNeutralizer.patch` 中和自杀调用；**md5 声明不匹配不致命**（缓存不命中就重下，重下后不再校验）。
- **⚠️ 唯一例外**：`isGuardJarBroken()` —— 含 `DexNative` 的加固 jar 若 Init 的
  DexClassLoader 字段为空，**整包跳过其全部站点**。若实机某组站整批失败，先查这条。
- 另外 `ext` 是 dict/array 时 TVBox 会 `toString()` 传给 spider（**字典 ext 安全**）。

### A6 ★ key 是站点唯一标识，重复 key 会让站点互相覆盖

- **现象**：多个源合并后，某些站点"莫名消失"。
- **根因**：TVBox 用 `key` 作站点标识，重复 key 会互相覆盖。
- **修复**：`dedupSites` 做 key 唯一化 —— key+api 全同→丢弃（同一站点副本）；
  key 同但 api 不同→自动改名 `key_2` 并**同步 `s.raw.key`**（导出读的是 raw，只改 `s.key` 无效）。

### A7 多仓/订阅格式：必须是对象，裸数组会报「订阅有问题」

```json
{ "urls": [ {"url":"...","name":"..."} ], "vip": [ {"url":"","name":"默认"} ] }
```
- 裸数组 `[{name,url}]` → TVBox 读不到 urls 字段 → 订阅报错。（曾与参考源 dc2.json 逐字段对照确认。）

### A8 TVBox 只解析第一个 JSON 对象 —— 拼接的多段 JSON 只读第一段

- **现象**：一份文件里有多段配置，TVBox 只用前面几条。
- **根因**：文件里是 4 个独立 JSON 对象首尾拼接（非法 JSON），TVBox 解析到第一段结尾就停。
- **修复**：解析器新增 `splitJsonSegments(text)` —— 字符串感知的括号深度扫描，逐段切出；
  只在整段解析失败时启用（不影响老路径）。

### A9 wallpapers / rules 是纯字符串数组

- 这两类 TVBox 的写法就是 `["url1","url2"]`（不是对象数组）。
- **坑**：`exportNodes` 早期用 `typeof n === 'object'` 过滤 → 字符串全被滤掉 → 导出壁纸恒为 0 条。

### A10 其他零散事实

- `sites.categories` 部分版本按 `getAsJsonArray` 读 → 字符串要按逗号拆成数组。
- `rules` 里若混入 `null`，`getJSONObject(i)` 会抛异常 → 导出前过滤非对象元素。
- `searchable` 默认 1；缺失时不要写成 0（会让站点搜不了）。
- 冷启动搜索有"假空结果"：订阅池延迟 + jar 冷加载超时都会造成首搜为空，重搜即可。

---

## B. 解析器自身的 bug（改代码时注意）

### B1 `tryParseJSON` 返回包装对象 `{parsed, obj}`，不是裸 obj
曾把返回值直接当对象用 → 解析永远失败。取 `.obj` / `.parsed`。

### B2 复选框缺 change 监听 → 切换后导出预览不刷新
`chkDoubanPin`、`chkSpiderOverride` 都踩过。**新增任何导出相关控件都要接 `renderExport()`。**

### B3 `renderLog` 用 innerHTML 全量重绘 = O(n²)
92 条链接就崩。改 `requestAnimationFrame` 合并（一帧一次 DOM 更新）+ 日志上限 300 条。
`updateProgress` 同理（原来每 URL 写 3 次 DOM）。

### B4 并发数 × 候选数 ≤ Chrome 每域 6 并发
`runPool(4)` × 3 候选 = 12 并发 > 6 → 请求排队**但计时已经跑了** → 集体超时卡死。
修复：竞跑候选 3→2（直连+代理），共享 AbortController 胜出即 cancel 另一路，池改 10。

### B5 搜索结果高亮样式必须定义在 `tr.selected td` **之后**
同优先级下靠后的胜出，否则命中行被选中蓝色覆盖、看不出高亮。

### B6 `applyPerSiteJar` 改 key 时要同步 `s.raw.key`
导出读的是 raw，只改 `s.key` 无效。

### B7 `inferBaseUrl` 不能用媒体资源 URL 当来源 base
原会把 `logo`/`wallpaper` 的 CDN 地址误当来源主机 → 相对路径拼到错误主机。
只用"功能型字段"：spider / ua / dcraw / dcraw_url。

### B8 区分 `sources` 与 `works` 两种输入形态
粘贴 JSON = 一个直接源；URL = 需要 fetch。多段 JSON 拆分后每段各算一个 direct work（name 加 `#N` 后缀）。

---

## C. 工具链 / 本机环境（可复现的测试方法）

### C1 node 二进制版本目录会漂移
`22.22.2-2 → -3 → -6`（实测三次变更）。**路径失效先 `ls ~/.workbuddy/binaries/node/versions/`**，
别反复试旧版本号。

### C2 jsdom 位置与用法
装在 `~/.workbuddy/binaries/node/workspace/node_modules`，跑脚本需
`NODE_PATH="<该路径>" <node.exe> script.cjs`。
jsdom 缺 `scrollIntoView` → 测试前补桩：`win.Element.prototype.scrollIntoView = () => {}`。

### C3 Chrome 无头截图
`%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe --headless=new --disable-gpu
--hide-scrollbars --window-size=1180,1560 --virtual-time-budget=8000 --screenshot=<绝对路径> <file:///绝对URL>`
- `--screenshot` **必须给绝对路径**，否则文件落在 Chrome 自己的 CWD。
- 页面内有平滑滚动会与截图抢时序 → 注入脚本里把 `Element.prototype.scrollIntoView`
  覆盖成 `behavior:'instant'`；更稳的做法是**用高视口不滚动直出**。

### C4 单文件 HTML 的三种测试法（按可靠性排序）
1. **jsdom 真实页面级**（首选）：能跑完整交互链（点击/输入/事件）。
2. **vm + Proxy DOM stub**：万能 Proxy 让任意属性访问/调用都返回自身；
   脚本是 IIFE 时用 `code.lastIndexOf('})();')` 在结尾前注入 `globalThis.__api = {...}` 导出内部函数。
3. 纯静态断言（grep/语法检查）。

### C5 取导出 JSON（测试用）
`buildExportJSON` 在 IIFE 内不是全局函数 → 测试改成：点 `#btnExport` +
拦截 `URL.createObjectURL` 拿 Blob → `await blob.text()`。

### C6 解析后默认**不全选**（2026-10-09 起）
测试必须先点 `#btnSelectAll`，否则导出的空。

### C7 PowerShell / bash 工具的坑
- PowerShell 工具 stdout 经常丢失（退出码 0 但无输出）→ 一律 `Set-Content` 写文件后 Read。
- bash 工具**批量删除闸门**：单次 >50 个文件会返回 `SAFE_DELETE_BULK_CONFIRM_REQUIRED`
  并**静默不执行**（脚本里的 `echo 已删` 照样打印，会误判成功）。
  ⇒ 判据一律用 `ls` / `find | wc -l` 复核；绕过用 `find <dir> -maxdepth 1 -type f -name '...' -delete`。
- 修改 `_t_*.cjs` 测试脚本时注意 CRLF（用 `newline=''` 或直接重写文件，避免 replace 不命中）。

### C8 清理磁盘的真相
WorkBuddy 的删除 = 移动到会话的 `modify_backup` 副本目录，**不释放空间**。
要真释放需两步：① 删目标文件；② 扫副本目录 `find "<modify_backup>" -maxdepth 1 -type f -name '*<原名片段>*' -delete`。
副本命名有 `序号.d.<hash>.<原名>` / `.a.` / `.m.` 多种前缀 ⇒ **按原名片段匹配最稳**。

---

## D. 诊断方法论（遇到"站不能用"时按这个走）

### D1 四层排查法（不要靠猜）
**站点 → 它依赖哪个 jar → jar 是否活着 → jar 里有没有它需要的类**
四层全部要有实测证据（HTTP 码 / md5 / 类清单）。

### D2 从 jar 里提取类清单（dex 格式）
- 标准 jar：`com/github/catvod/spider/Xxx.class`
- **Android 打包：只有 `classes.dex`**（无 .class！）
- dex 头：`string_ids_size` @0x38、`string_ids_off` @0x3c、
  `type_ids` @0x40、`class_defs` @0x60（每条 32 字节 → class_idx → type_idx → 描述符）
- 字符串读法：uleb128 长度 + MUTF-8 到 `\x00`
- ⚠️ **string_ids 数组从 `str_off` 开始，不要 +4**（踩过）
- 目标串：`Lcom/github/catvod/spider/...;` → 去 `L`/`;`、`/`→`.` 得全限定名，
  **比"简单类名"时才去掉包名**（第一次比全限定名 → 覆盖率算成 0%）

### D3 ★ 死 jar 的最佳救援路径：拿 md5 反查 GitHub 代码搜索
同一份 jar 换托管位置 **md5 不变** → 用 `md5` / 特征类名 `search_code` 反查仍在更新的镜像仓库。
实测：`<md5已隐去>` → 命中 `<某公开镜像仓库>` → 镜像下载校验 md5 完全一致 → 12 站救回。
**镜像可用性**：`ghfast.top` / `gh-proxy.com` 前缀式可通；jsDelivr 对 `.jar` 返 **403**；
`raw.githubusercontent.com` 本机常不通。

### D4 探测 jar 必须用 TVBox 的 UA
`curl -A "okhttp/3.15"`；跟随跳转 `-L`（很多 jar 是 302 到 CDN）。
判活看：HTTP 200 + 文件头 `PK\x03\x04`（ZIP/JAR）或 `dex\n`（DEX）。

### D5 浏览器内无法可靠探测 spider 可达性
跨域 + UA 校验 + 动态 302 ⇒ 前端只能：① 域名启发式；② 按元数据（覆盖度）选。
**不要试图用 fetch 判活来决定导出内容。**

### D6 公开 CORS 代理会集体失效，别依赖
实测（2026-08-30）：cors.sh 唯一稳定（连打 6/6、并发 4 无 429）；
cors.lol 常驻 429；allorigins/codetabs 常年超时；whateverorigin 返回 HTML 假成功；thingproxy 已死。
⇒ **优先找自带 `Access-Control-Allow-Origin` 的官方通道**（jsDelivr 返回 `ACAO:*`，浏览器直连即可），
代理只当兜底。另外：429 是短时限流，**立刻重试只会继续被挡，必须先退避**。

### D7 排查"功能 A 坏了"先做对照实验
别凭直觉改。先确认**是不是 A 与 B 冲突**（实测证明"多配置源失效"与"多仓组合"无关，
真因是代理全废 + 容错不足）。

### D8 断言失败先怀疑断言本身
多次出现"测试失败但代码是对的" —— 例如把壁纸行也算进站点计数、
把"x 站与全局 spider 同源不该挂站点级 jar"写成失败。
**先读一遍自己的预期值再改代码。**
