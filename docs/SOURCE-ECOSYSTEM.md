# 配置源生态档案（SOURCE-ECOSYSTEM）

> ⚠️ **本文件含具体资源地址（jar / 镜像 / 令牌），不得公开托管，只能放私有仓库。**
> 内容：TVBox 源生态机制 + 具体诊断案例 + 可复用脚本片段。
> 机制层面的结论已提炼进 `PITFALLS.md`，这里是**带具体数据的案例集**。

---

## 一、spider jar 生态机制

### 1.1 三种 jar 形态（都要能识别）

| 形态 | 特征 | 识别方式 |
|---|---|---|
| 标准 jar | 含 `com/github/catvod/spider/Xxx.class` | zip 内 .class 文件名 |
| Android 打包 | **只有 `classes.dex`**（无 .class） | zip 内 classes.dex → 解析 dex |
| 图片伪装 | URL 以 `.jpg`/`.png` 结尾 | 文件仍是 ZIP/DEX，看文件头 |

- `img+` 前缀写法：`img+http://...jpg` —— App 端会自动剥掉前缀，通过图片通道下发 jar。
  解析器需在全链路（判定/归一化/导出/审核/探测）都剥离该前缀再处理。
- **扩展名不说明任何事**：`.php` / `.jpg` / `.txt` 都可能是 jar。

### 1.2 类名与加载规则

- `csp_Xxx` 中 `csp_` 是 **api 方法前缀**，不是类名的一部分：
  `csp_Bili` → 类 `Bili` → 加载 `com.github.catvod.spider.Bili`。
- 同理 `js_` / `py_` 分别对应 JS / Python 爬虫。
- 站点级 `sites[].jar` 会走**独立的 DexClassLoader**（与全局 spider 隔离）。

### 1.3 加固 jar（含 DexNative）

- 部分 jar 带 native 加固（assets 里有 `.so`，如 `wexguard_v7.so`）。
- App 判定：`isGuardJarBroken()` —— 含 `DexNative` 的加固 jar 若 Init 的 DexClassLoader
  字段为空 → **整包跳过其全部站点**（不是单站失败，是整批）。
- ⇒ 加固 jar **不可替换**（换别的 jar 会丢类或触发防护）。
- 实测带加固的：<某对象存储> 系（Wex*Guard / NewDouBanGuard）、<某jar> 系（ftyguard）。

### 1.4 UA 校验（最容易误判的坑）

实测案例 `http://<某IP:端口>/vip/jar/<某jar端点>`：

| 请求 UA | 返回 |
|---|---|
| 浏览器 UA | HTML 广告页（31KB） |
| `okhttp/3.15` | 真 jar（2MB / 180 类） |

⇒ 任何 jar 判活/下载都要用 `-A "okhttp/3.15"`。**别用浏览器探测结果下结论。**

---

## 二、诊断脚本片段（可复用）

### 2.1 探测 jar 是否活着（curl）

```bash
# 判活三件套：跟随跳转 + okhttp UA + 看文件头
curl -skL -m 45 -A "okhttp/3.15" -o _j.bin -w "HTTP %{http_code} size=%{size_download}\n" "<jar_url>"
md5sum _j.bin            # 与配置声明的 ;md5; 比对
head -c 4 _j.bin | xxd   # 504b0304 = ZIP/JAR；6465780a = DEX；3c68746d = HTML(广告页)
```

### 2.2 从 jar 提取类清单（Python）

```python
import zipfile, struct, io

def dex_classes(buf):
    """从 classes.dex 提取定义类名（全限定）"""
    sids_n, sids_o = struct.unpack_from('<II', buf, 0x38)   # string_ids_size / off
    tids_n, tids_o = struct.unpack_from('<II', buf, 0x40)   # type_ids
    cdefs_n, cdefs_o = struct.unpack_from('<II', buf, 0x60) # class_defs
    def get_str(i):
        off = struct.unpack_from('<I', buf, sids_o + i*4)[0]
        j = off
        while buf[j] & 0x80: j += 1      # uleb128 长度
        j += 1
        return buf[j:buf.index(b'\x00', j)].decode('utf-8', 'replace')
    out = []
    for k in range(cdefs_n):
        ci = struct.unpack_from('<I', buf, cdefs_o + k*32)[0]      # class_idx
        ti = struct.unpack_from('<I', buf, tids_o + ci*4)[0]       # type_idx
        d = get_str(ti)
        if d.startswith('L') and d.endswith(';'):
            out.append(d[1:-1].replace('/', '.'))
    return out

data = open('target.jar', 'rb').read()
z = zipfile.ZipFile(io.BytesIO(data))
full = set()
for n in z.namelist():
    if n.endswith('.dex'):
        full |= set(dex_classes(z.read(n)))
simple = {c.split('.')[-1] for c in full}   # ★ 比类名要去包名！
print('类数:', len(simple))
```

**坑**：① `string_ids` 数组从 `str_off` 开始，不要 +4；② 比对 csp_Xxx 时要同时
去 `csp_` 前缀和包名。

### 2.3 用 md5 反查 GitHub 镜像（救死 jar 的标准流程）

```
1. 从失效配置里取 md5（;md5;xxxx 形式）
2. GitHub code search: "xxxx"  → 找到仍在更新的仓库
3. 拼镜像地址下载 → md5sum 校验必须完全一致
4. 校验通过后才写进配置
```

实测可用镜像前缀：`https://ghfast.top/<raw_url>`、`https://gh-proxy.com/<raw_url>`
（jsDelivr 对 `.jar` 返 **403**；`raw.githubusercontent.com` 本机常不通）。

### 2.4 后端站点探活（判断"是配置问题还是上游挂了"）

```python
# 对站点 ext 的 URL 直接 GET，看状态码与响应体
# 200 + 内容是 JSON/播放列表 → 活；404/522/超时 → 上游问题，配什么 jar 都救不了
```

---

## 三、具体案例：41 站配置的三轮救治（2026-08-27 ~ 10-09）

### 3.1 案例背景

桌面 `data.json`，41-47 个影视站，**全部是 `type:3` 的 `csp_` 站点**，
分属 4 个不同的 jar（因为是多份配置拼接而来）。

### 3.2 第一轮（08-27）：41 站全灭

| 层 | 问题 | 处理 |
|---|---|---|
| 全局 spider | 404（`xyq254245/xyqonlinerule` 的 spider.jar 不存在） | 换成含豆瓣类的 HTTPS jar |
| jar-D | `<某托管域名>/jar/<某jar>` 404（桶清空） | 8 站靠 md5 反查救回 |
| 无 jar 站 | 2 站（橘汁/永乐）只能依赖已 404 的全局 spider | 注入可用 jar |

**md5 反查过程**：
- <某域名>的多个变体（space/dev/run/cfd/<某托管域名> 等 42 组合）全部返回 HTML 404 页 → 桶确实清空。
- GitHub code search `csp_SixVGuard csp_Dm84Guard` → 找到 `<某公开镜像仓库>` 的 <某公开配置>，
  其 `"spider": "./jar/<某jar>;md5;<md5已隐去>"` **md5 完全一致**。
- 直连 raw 被墙 → **jsDelivr 可通**：`https://cdn.jsdelivr.net/gh/<某仓库>@master/jar/<某jar>`
  → 1.1MB，md5 与原值一致，dex 类 8/8 全中。

### 3.3 第二轮（08-30）：per-site jar 注入

- 源 spider 对本源站点的类覆盖率：源A 97.5%、源B 100%，汇总 98.3%。
- 3 源合并（195 个依赖站点）：修复前 63.6% → 修复后 **99.5%**。
- 实际修复桌面文件：977 个站点补上 jar，61 个仍缺（覆盖率 52.2% → **96.9%**）。

### 3.4 第三轮（10-09）：拼接 JSON + 死 jar 换镜像

**根因两层**：
1. 文件是 **4 个独立 JSON 对象首尾拼接**（各有 exportTime / spider / 站点）→ TVBox 只读第一段。
2. 4 段各带不同 jar，**全局 spider 只有 1 个槽位** → 必须靠站点级 jar 才能共存。

**4 段原始数据**：

| 段 | 导出时间 | 自带 spider | 站点数 | 站点用到的类 |
|---|---|---|---|---|
| ① | 14:19:54 | `<某对象存储>/.../xxx.jpg;md5;`（伪装 jpg） | 16 | csp_WexXxxGuard / csp_NewDouBanGuard |
| ② | 14:24:04 | `<某IP:端口>/vip/jar/<某jar端点>` | 7 | csp_AppRJ / csp_Gz360 / csp_JianPian |
| ③ | 14:31:13 | `<某源域名>/jar/<某jar>;md5;` | 6 | csp_BttwooGuard / csp_JpysGuard |
| ④ | 14:43:21 | `<某域名:端口>/spider.jar;md5;<md5已隐去>` | 18 | csp_AppDrama / csp_Gulu / csp_SaoHuo |

**合并结果**：47 站去重为 41 站；jar 归属 16 <某对象存储> / 7 <某jar> / 6 <某jar> / 12 随全局。

**"只有 25 个有效"的根因**：段 4 的 `<某域名:端口>/spider.jar` **整站 404**
（站点根目录也是 404 页）→ 12 站全灭。

**救援过程（md5 反查）**：
- 目标 md5 `<md5已隐去>`
- GitHub code search 命中 `<某公开镜像仓库>`（jsm.json / dianshi.json / 镜像/api.json 三处引用同一 md5）
- 镜像地址：`https://ghfast.top/<原始raw地址>`
  （备用 `gh-proxy.com` 同 md5 可用）
- 下载校验：1,859,860 B / md5 完全一致 / 2262 个类 / **含 Gulu + FengYe** → 41/41 类全覆盖

**顺带修复**：`📺热播┃多线`、`🐻剧圈┃多线` 的 ext 里被前置了一个失效主机
（`<某源域名>`，404）。对照作者官方 `<某公开镜像仓库>/<某公开配置>` 和 GitHub 上 342 个同源仓库，
**全都是裸令牌形式**（jar 内部自带 base host）→ 已改回裸令牌。

**唯一预期失败的站**：`骚火 • 影视` —— 后端 `<某站点域名>` 返回 Cloudflare 522（源站挂了）。

### 3.5 各 jar 实测数据（2026-10-09）

| jar | 实测 | 类数 | 备注 |
|---|---|---|---|
| <某对象存储>（伪装 .jpg） | HTTP 200 / 1,000,156 B / md5 <md5前缀…> ✅ | 116 | 含 assets/*.so（**加固，不可换**） |
| <某jar端点> | 302 → <某自建Git服务> `<某jar>` / 2,081,666 B | 2339 | 超大合集（4258 类），**无 Gulu/FengYe** |
| <某jar>（<某源域名>） | HTTP 200 / 1,255,425 B / md5 <md5前缀…> ✅ | 57 | 带 ftyguard native |
| 镜像来源（ghfast.top） | HTTP 200 / md5 <md5前缀…> ✅ | 2262 | 含 Gulu + FengYe |

**单 jar 覆盖率上限**：<某对象存储> 16、<某jar> 6、<某jar> 17、镜像 19（最高 19/41）
⇒ **站点级 jar 是硬需求，不能简化成单 spider。**

---

## 四、历史踩坑索引（按案例速查）

| 症状 | 去 PITFALLS 看 | 本案实例 |
|---|---|---|
| 配置解析失败 | A1 | parses.ext 58 个字符串 |
| jar load err | A2 | `./OS.jar` 相对路径 + 源B 死链 |
| 搜索为空但代理正常 | A3 | 三个源连续踩（<某托管域名> / 源A / 源B） |
| 部分站点消失 | A6 | 荐片 key 冲突 |
| 订阅有问题 | A7 | 裸数组多仓 |
| 只有前面几条能用 | A8 | 4 段拼接 JSON |
| 壁纸导出 0 条 | A9 | 字符串数组被过滤 |
| 类覆盖率算成 0% | A4/D2 | csp_ 前缀 + 全限定名 |
| 好 jar 被判死 | D4/1.4 | <某jar端点> 的 UA 校验 |
| 死 jar 救不回 | D3/2.3 | 两次靠 md5 反查成功 |

---

## 五、长期观察（生态事实）

1. **没有永久稳定的 jar 地址**。任何"默认兜底"都会失效（<某jar> 就失效过）。
   长期方案 = 给出可更换的默认 + 清晰替换指引，而不是追求"永不失效的默认"。
2. **上游站点后端会大面积死亡**。ext 是 URL 的站点死亡率最高（实测 8 个 URL 型 ext 死 6 个）。
   这类失败与配置无关，配什么 jar 都救不了 → 只能换源。
3. **镜像镜像再镜像**：同一个 jar 会在多个仓库间流转（md5 不变），
   所以"md5 反查"比"记住地址"可靠得多。
4. **公共 CORS 代理会集体失效**（2026-08-30 实测全部阵亡）。
   自带 `ACAO:*` 的官方通道（jsDelivr）才是可靠路径。
