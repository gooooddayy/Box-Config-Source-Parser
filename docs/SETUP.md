# 换电脑上手指南（SETUP）

> 新电脑 / 新会话从这里开始，10 分钟恢复完整开发能力。
> 本指南基于 2026-10-09 的实测数据（通道稳定性、环境坑）写成。

---

## 0. 新会话开场怎么说

对 WorkBuddy 说一句：

> 从 GitHub 拉取 Box-Config-Source-Parser 项目到工作区，我要改 XXX

AI 会：拉文件 → 读 `docs/PITFALLS.md` 避坑 → 等你提具体需求。

---

## 1. 拉取文件（三条通道，按稳定性选）

实测（2026-10-09，本机网络）：

| 通道 | 稳定性 | 用法 |
|---|---|---|
| **GitHub API** | ✅ 稳定（首选） | `GET https://api.github.com/repos/gooooddayy/Box-Config-Source-Parser/contents/<路径>?ref=main`（公开仓库匿名可读；返回 base64，解码即得） |
| **Pages 域名** | ✅ 稳定（备用） | `https://gooooddayy.github.io/Box-Config-Source-Parser/<路径>` |
| raw.githubusercontent.com | ⚠️ 时通时断 | 3 次测试通 1 次；失败就换上面两条 |

**注意**：`git clone` 在部分网络环境下握手失败（`schannel: failed to receive handshake`）——
**不用 clone，直接走 REST API** 就够了（仓库就几个文本文件）。

拉取后自检：`index.html` 应约 120KB / 2286 行，含 `</html>` 结尾，
跑一次 `node --check`（见下）确认完整。

---

## 2. 搭验证环境

改代码后必须过三关（方法论详见 `docs/PITFALLS.md` C 部分）：

### ① 语法检查
抽出 `<script>` 内容跑 `node --check`。
- node 路径：WorkBuddy 管理版本在 `~/.workbuddy/binaries/node/versions/<版本号>/node.exe`
- ⚠️ **版本目录会漂移**（实测 22.22.2-2 → -3 → -6），先 `ls versions/` 看有哪些再用

### ② jsdom 功能测试
- jsdom 装在 `~/.workbuddy/binaries/node/workspace/node_modules`，跑脚本时设 `NODE_PATH`
- 新机器没有就装：`cd <workspace目录> && node install jsdom`（装进 managed workspace，别装全局）
- 已知桩需求：`Element.prototype.scrollIntoView` 需补空函数；取导出结果要点 `#btnExport` + 拦截 `URL.createObjectURL` 拿 Blob

### ③ Chrome 无头截图
- Chrome 位置各机不同，先找：`$LOCALAPPDATA/Google/Chrome/Application/chrome.exe` 或 `C:\Program Files\Google\Chrome\...`
- `--screenshot` 必须给**绝对路径**（否则落在 Chrome 自己的目录）
- 页面有平滑滚动时，注入脚本里把 `scrollIntoView` 覆盖成 instant，或用高视口直出

---

## 3. 开发循环

```
读 docs/PITFALLS.md A 部分（配置格式机制 —— 不读会反复踩同一个坑）
  ↓ 改代码（对照 docs/ARCHITECTURE.md 的模块地图与修改指南）
  ↓ 验证：语法 → jsdom 功能测试 → 截图目视
  ↓ 推送（见下节）
  ↓ Pages 自动更新（约 35 秒）
```

---

## 4. 推送回 GitHub

### 4.1 前置：token

GitHub 网页 → Settings → Developer settings → Personal access tokens → Generate new token，
勾 `repo` 权限即可。

⚠️ token = 账号写权限：只发到对话里、用完就撤销重新生成；**绝不写进任何文件**（脚本从环境变量读）。

### 4.2 推送前必做的两件事

1. **查最新 commit**（防覆盖）：
   `GET /repos/gooooddayy/Box-Config-Source-Parser/commits?sha=main&per_page=1`
   ⚠️ **用户会自己在 GitHub 网页上直接编辑文件**（实测发生过）。不查就推 = 覆盖用户的修改。
   必须以最新 commit 的 tree 为 base_tree 修改。
2. **合规扫描**：新增/修改内容过一遍下方红线清单。

### 4.3 推送方式：Git Data API 单 commit

```
POST /git/blobs          （每个文件一个，base64）
POST /git/trees          （base_tree = 最新 commit 的 tree，只放改动文件 → diff 干净）
POST /git/commits        （parents = 最新 commit）
PATCH /git/refs/heads/main
```

**🔴 换行符坑（实测踩过）**：Python `read_text()` 会把 CRLF 转成 LF，
`write_text()` 默认又把 LF 写回成 CRLF（Windows）——两头一凑就产生
"整个文件都变了"的假 diff（2285 行全变，实际只改了 9 行）。
**写回时必须显式 `newline=''`，或直接操作 bytes。**
推送后用 `compare` 接口看 diff 行数，确认只有预期改动。

**force 更新**：只在修正自己刚推错的 commit 时用（`PATCH ref` 带 `force: true`，
parent 指向前一个正确 commit），不要覆盖别人的提交。

### 4.4 部署验证

Pages 部署约 35 秒：
`GET /repos/gooooddayy/Box-Config-Source-Parser/pages/builds/latest`
→ `status: built` 且 commit 一致即完成，再到线上抽查内容。

---

## 5. 合规红线（公开仓库，改任何内容都要守）

| 禁止 | 允许 |
|---|---|
| ❌ jar 完整地址、`;md5;` 真实值 | ✅ 技术术语（TVBox / jar / spider / csp_ / dex）—— 代码注释与 docs 里保留是有意的 |
| ❌ 特定资源域名、镜像仓库路径 | ✅ 通用基础设施裸域名（GitHub / jsDelivr / ghproxy 系列 / 公共 CORS 代理） |
| ❌ 具体站点名（上游站点的显示名） | ✅ 功能名（如豆瓣置顶 —— 合法网站名 + 用户实际要用的开关） |
| ❌ 在标题/描述里加内容行业指向（保持「配置解析/合并」定位） | ✅ 中性表述：README 用"配置解析/合并"，界面提示用"主程序""条目" |

每次推送前跑一遍敏感扫描（正则匹配上述禁止项），命中即改。

---

## 6. 关联项目（勿混）

- `gooooddayy/boxURL` —— 放配置成品 data.json + JSON 查看器。
  **与本项目相互独立，不要互相覆盖。**

---

## 7. WorkBuddy 环境备忘（换电脑相关）

- **用户级记忆**（跨项目偏好）存于本机 `~/.workbuddy/user-<id>/MEMORY.md`，换电脑**不会自动同步**；
  云端画像（服务端管理）跟账号走，核心偏好不丢
- **会话历史 jsonl 会无限膨胀**（长项目实测 270MB+），是界面卡顿头号原因 → 阶段性开新会话
- 删除 = 移动：WorkBuddy 会把删除的文件备份到会话 `modify_backup/`，大清理后要回头清副本才真正释放空间
