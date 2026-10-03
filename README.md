EQ 旗舰部署版 · 2.3.0 修复 + 能力补全版

基于 Cloudflare Workers 的代理服务端。`2.2.0` 修复了四类故障；`2.3.0` 在此基础上补齐了
**多协议（Trojan / Shadowsocks）、UDP-DoH、subconverter 订阅转换**，并把 **KV 命名空间名称写死**
以便按名称创建绑定。

完整的证据、根因与回归测试结果见 **[AUDIT-REPORT.md](./AUDIT-REPORT.md)**。

---

## 一、2.3.0 新增能力

| 编号 | 能力 | 实现方式 | 验证结果 |
|---|---|---|---|
| ADD-01 | **Trojan over WebSocket** | 标准 trojan 头 `hex(SHA-224(password)) + CRLF + cmd + ATYP + ADDR + PORT + CRLF`；SHA-224 为纯 JS 实现 | 握手+转发成功，错误口令被拒；SHA-224 与 Node 内置实现 137 例逐例一致 |
| ADD-02 | **Shadowsocks AEAD over WebSocket** | SIP007：`EVP_BytesToKey(MD5)` 主密钥 + `HKDF-SHA1(info="ss-subkey")` 会话子密钥 + AES-GCM + 12 字节小端 nonce 计数器 | aes-128-gcm / aes-256-gcm 均完成「加密→服务端解密→回程加密→客户端解密」闭环 |
| ADD-03 | **UDP（DNS over HTTPS）** | VLESS/Trojan 的 UDP 报文按 2 字节长度分包；`:53` 的 DNS 查询转 DoH 请求并回封响应 | 实测返回真实 DNS 应答（QR=1，answers=2）；非 53 端口明确拒绝 |
| ADD-04 | **内置 subconverter** | 支持 `clash / singbox / surge / quanx / loon / mixed / raw`，也可转发外部 SUBAPI，失败自动回退 | 22 项订阅测试全通过 |
| ADD-05 | **KV 诊断** | `/admin/kv` 暴露绑定名、读写探测结果；面板顶部在未绑定 KV 时弹橙色告警条 | 已验证 `bound/write/healthy` 三态 |
| ADD-06 | **测试脚本随包交付** | `tests/` 下 4 个脚本，可在本地重跑全部结论 | 见「七、自测」 |

---

## 二、2.2.0 修复清单（保留）

| 编号 | 问题 | 修复 |
|---|---|---|
| FIX-01 | 日志加载失败（`/admin/log.json` 返回 HTML） | 补齐日志接口 + 全链路日志落盘 |
| FIX-02 | 选择性配置保存后刷新丢失 | 补齐 `cf.json` 写入、`tg.json`、`init`、`getCloudflareUsage`、`version`；未知接口返回 JSON 404 |
| FIX-03 | 密码无法修改 | 新增 `/admin/password` + 面板改密弹窗 + 默认口令强制改密 |
| FIX-04 | 保存静默失败（返回 200 HTML，前端误判成功） | 后台接口一律返回 JSON |
| FIX-05 | 未绑定 KV 时保存直接 500 | 降级内存存储并明确提示 |
| FIX-06 | 默认配置字段与前端不匹配 | 补齐全部字段 |
| FIX-07 | 订阅节点地址每次随机变化 | 改用后台优选节点，输出恒定 |
| FIX-08 | Clash 规则 `MATCH,DIRECT`（全直连） | 改为 `MATCH,PROXY` |
| FIX-09 | 反代回传上游 Cookie | 删除上游 `Set-Cookie` |
| FIX-10 | Cookie 绑定 UA，换浏览器掉线 | 与 UA 解耦 |
| FIX-11 | 登录可被爆破 | 每 IP 5 分钟 8 次失败后限流 |
| FIX-12 | 缓冲区复用可能串包 | 首包拷贝后再归还内存池 |
| FIX-13 | 缺 `/version`、`/admin/status` | 补齐 |

---

## 三、部署：KV 名称已写死

> **固定名称（不要改）**
> - 命名空间名称（title）：`edgetunnel-kv`
> - 绑定名（binding）：`KV`

### 方式一：Wrangler CLI（推荐，一键）

```bash
npm install                 # 安装 wrangler（也可直接用 npx）
npx wrangler login          # 首次需要登录
node setup-kv.mjs           # 自动创建 edgetunnel-kv 并把 id 回填到 wrangler.toml
npx wrangler deploy
```

`setup-kv.mjs` 的三种用法：

| 命令 | 作用 |
|---|---|
| `node setup-kv.mjs` | 自动创建 `edgetunnel-kv`；若已存在则尝试复用，并回填 id |
| `node setup-kv.mjs <32位ID>` | 已有命名空间，直接把 id 写回 `wrangler.toml` |
| `node setup-kv.mjs --check` | 只检查 id 是否已配置好 |

### 方式二：Cloudflare 控制台手动创建

1. **Workers & Pages → KV → 创建命名空间**，名称填 `edgetunnel-kv`
2. 复制命名空间 ID
3. 运行 `node setup-kv.mjs <ID>`（或手动替换 `wrangler.toml` 末尾的 `id`）

### 方式三：在线编辑器粘贴 `_worker.js`

> ⚠️ 这种方式**不会自动创建 KV 绑定**，配置只能落在内存里，冷启动后丢失。
> 你之前遇到的「保存后刷新就没了」「貌似连不上 KV」，大概率就是这个原因。

若必须用这种方式，请在控制台补齐绑定：
**Worker → 设置 → 变量 → KV 命名空间绑定 → 变量名填 `KV`（必须叫这个名字）→ 选择 `edgetunnel-kv`**

绑定完成后，登录后台访问 `/admin/kv` 应返回 `"healthy": true`；未绑定时面板顶部会有橙色告警条。

---

## 四、多协议怎么用

`wrangler.toml` 里控制订阅输出哪些协议：

```toml
PROTOCOLS = "vless,trojan,ss"     # 只要 VLESS 就改成 "vless"
TROJAN_PASSWORD = "你的trojan口令"
SS_METHOD = "aes-128-gcm"         # 或 aes-256-gcm
SS_PASSWORD = "你的SS口令"
SS_PATH = "/ss"                   # Shadowsocks 独立路径
```

| 协议 | 客户端 | 关键点 |
|---|---|---|
| VLESS | v2rayN / v2rayNG / sing-box / Clash | 主路径（`PATH`，默认 `/`），TLS + WS |
| Trojan | trojan-go / Clash / sing-box | 主路径，口令即 `TROJAN_PASSWORD`，`type=ws` |
| Shadowsocks | Shadowsocks + v2ray-plugin | 路径必须是 `/ss`，加密方式 `aes-128-gcm`，插件 `v2ray-plugin;mode=websocket;path=/ss;host=<你的域名>;tls` |

> Shadowsocks 用独立路径，是因为 SS 报文是纯密文、没有协议特征头，
> 无法与 VLESS/Trojan 在同一个路径上自动区分。

---

## 五、订阅格式

在订阅地址后加 `target=` 参数（也可在面板 `/admin/sub-links` 拿到全部链接）：

| 地址 | 输出 |
|---|---|
| `/sub` | base64 混合链接（默认，通用） |
| `/sub?target=clash` | Clash / Mihomo YAML |
| `/sub?target=singbox` | sing-box JSON |
| `/sub?target=surge` | Surge 配置片段 |
| `/sub?target=quanx` | Quantumult X 节点行 |
| `/sub?target=loon` | Loon 配置片段 |
| `/sub?target=raw` | 明文链接列表 |
| `/sub?target=<其它>` | 交给外部 subconverter（需配置 `SUBAPI`） |

也支持按 UA 自动识别（Clash / sing-box / Surge / Quantumult / Loon 会自动拿到对应格式）。

**外部 subconverter（可选）**：在 `wrangler.toml` 填 `SUBAPI = "https://你的subconverter/sub"`，
或后台 `订阅转换配置.SUBAPI`。外部调用失败会自动回退内置转换，不会让订阅挂掉。
用 `?convert=api` 强制走外部，`?convert=local` 强制内置。

---

## 六、UDP 说明（重要）

Cloudflare Workers **没有 UDP socket**，所以：

- ✅ **支持**：目标端口 `53` 的 DNS 查询 —— 转成 DoH 请求，把应答封回 UDP 报文返回
- ❌ **不支持**：其它 UDP 端口（游戏、QUIC、WireGuard 等）—— 会被明确拒绝并记日志，不做静默丢弃

DoH 上游可换：`DOH_URL = "https://dns.google/dns-query"` 或 `https://1.1.1.1/dns-query`。
自检：登录后访问 `/admin/doh-test`。

---

## 七、自测

包内 `tests/` 目录下 4 个脚本，可复现本报告的全部结论：

```bash
# 1) SHA-224 正确性（无需起服务）
node tests/sha224.test.mjs

# 2) 协议互操作（需先起本地服务，并准备 wrangler 环境）
npx wrangler dev --local --port 8792
PORT=8792 node tests/protocol.test.mjs       # 需 Node 22+（用内置 WebSocket）

# 3) 订阅转换与新增接口
PORT=8794 ADMIN_PW=你的后台密码 python3 tests/subscription.test.py

# 4) 2.2.0 全量回归（54 项，带 KV / 不带 KV 两种场景）
PORT_A=http://127.0.0.1:8794 PORT_B=http://127.0.0.1:8795 python3 tests/regression.py
```

最新一次执行结果：`SHA-224 137/137`、`协议互操作 7/7`、`订阅与接口 22/22`、`2.2.0 回归 54/54`。

---

## 八、环境变量

| 变量 | 说明 | 默认 |
|---|---|---|
| `UUID` | VLESS 节点 UUID | `d342d11e-...` |
| `ADMIN` | 后台密码 | 同 UUID |
| `KEY` | 快速订阅密钥 | `EdgeTunnel-Secret-Key` |
| `FALLBACK_SITE` | 伪装反代站点 | `https://www.bing.com` |
| `PATH` | VLESS/Trojan 的 WS 路径 | 空（根路径） |
| `PROXYIP` | 出站备用 ProxyIP 池 | 官方优选列表 |
| `PROTOCOLS` | 订阅输出协议 | `vless,trojan,ss` |
| `TROJAN_PASSWORD` | Trojan 口令 | 同 UUID |
| `SS_METHOD` | SS 加密方式 | `aes-128-gcm` |
| `SS_PASSWORD` | SS 口令 | 同 UUID |
| `SS_PATH` | SS 的 WS 路径 | `/ss` |
| `DOH_URL` | DoH 上游 | Cloudflare |
| `SUBAPI` | 外部 subconverter | 空（用内置） |
| `PAGES_BASE` | 面板前端静态源站 | `https://edt-pages.github.io` |
| `REQUIRE_SUB_TOKEN` | 订阅是否强制带 token | `false` |

---

## 九、自检接口

| 地址 | 用途 |
|---|---|
| `/version` | 版本号、存储模式、是否需改密 |
| `/admin/status` | 同上（需登录） |
| `/admin/kv` | **KV 绑定状态与读写探测结果** |
| `/admin/protocols` | 当前启用的协议、口令、SS 方法、UDP/DoH 配置 |
| `/admin/sub-links` | 各格式订阅链接 |
| `/admin/doh-test` | DoH 连通性自检 |
| `/admin/log.json` | 运行日志 |

默认后台密码：`d342d11e-d424-4583-b36e-524ab1f0afa4`（公开值，登录后请先修改）。

---

## 十、目录说明

```
_worker.js            主程序（部署只需要这一个文件）
wrangler.toml         Wrangler 配置（KV 名称已写死）
setup-kv.mjs          KV 一键创建/回填脚本
package.json          项目描述
AUDIT-REPORT.md       审计报告：问题清单、证据、回归测试
README.md             本文件
tests/
  sha224.test.mjs     SHA-224 与 Node 内置实现交叉验证
  protocol.test.mjs   VLESS / Trojan / SS / UDP-DoH 互操作测试
  subscription.test.py 订阅多格式 + 新增接口测试
  regression.py       54 项全量回归（带 KV / 不带 KV）
frontend/
  login.html          登录页
  admin.html          管理后台（已注入改密功能）
  noKV.html           未绑定 KV 提示页
  noADMIN.html        未设置管理员密码提示页
config_guide.md       客户端配置指引
clash-meta.yaml       Clash Meta 配置模板
sing-box.json         sing-box 配置模板
```

> `frontend/` 下的页面默认由 `PAGES_BASE` 源站提供（与官方原版行为一致）。
> 直接修改本地 `admin.html` **不会生效**（除非自托管并改 `PAGES_BASE`）——
> 这也是改密组件采用「回源时动态注入」的原因。

---

## 十一、已知限制（不掩饰）

1. **Shadowsocks 2022（SIP022）未实现**。其强制要求 BLAKE3 `derive_key`，Workers 无原生支持；
   纯 JS 实现无法与真实客户端做互操作验证，与其交付一个连不上的功能，不如明确不提供。
   本次提供的是 **Shadowsocks AEAD（SIP007）**，这也是 v2ray-plugin 走 WS 时实际使用的格式。
2. **Surge / Quantumult X / Loon 三种格式按公开文档语法生成**，未做真机导入验证；
   Clash 与 sing-box 已做结构化解析验证（YAML/JSON 可解析、字段完整）。
   若某个客户端导入异常，请配置 `SUBAPI` 走成熟的 subconverter。
3. **UDP 仅 53 端口**，原因见第六节，属平台限制而非实现缺陷。
4. 面板前端仍默认从第三方 Pages 回源（与原版一致）；需要完全自控请设置 `PAGES_BASE` 自托管。
EdgeTunnel EQ 新版 全面审计报告与修复说明

> 审计对象：新版 `LikenZiCEn79832/EQ` vs 原版 `cmliu/edgetunnel`
> 审计方式：源码静态审计 + 本地 workerd（wrangler dev --local）真实请求复现
> 原则：**只记录有证据的结论**；无法复现或无法确认的，单独标注为「静态风险」或「已排除」

---

## 一、审计基线（可复现）

| 项目 | 原版 | 新版（审计对象） |
|---|---|---|
| 仓库 | `cmliu/edgetunnel` | `LikenZiCEn79832/EQ` |
| 分支/提交 | `main` @ `af4f9837e1843e34159018713bc8749ccec3004d` | `main` @ `772f3abbe4536e20ed172c770df4f1dd4c156751` |
| 主文件 | `_worker.js` 6646 行 / 288 KB（sha256 前缀 `dc7428d0651f7b07`） | `_worker.js` 909 行 / 33 KB（sha256 前缀 `3d6aa70b0dbb804e`） |
| 备注 | — | 仓库内只有 `README.md` + `EQ-fixed问题修复.zip`，代码在压缩包内 |

用户提供的附件（`EQ-fixed问题修复.zip`）下载被安全策略拦截（ssrf_blocked），
因此改用仓库内**同名压缩包**作为新版代码来源，二者应同源。

---

## 二、用户报告的 4 个已知故障 —— 全部复现并定位根因

### 故障 1：日志加载失败 【已复现】

**证据**：本地 workerd 实测（绑定 KV 与不绑定 KV 结果一致）

```
GET /admin/log.json  (带合法 Cookie)
  -> 200
     content-type: text/html; charset=utf-8
     body: <!DOCTYPE html><html lang="zh-CN">...<title>管理后台</title>...
```

**根因**：`_worker.js` 中 `/admin/*` 只有 5 个分支（config.json / ADD.txt / cf.json / check / 兜底页面），
**没有 `log.json` 分支**，请求落到第 401 行兜底：

```js
// 第 400-401 行
// 5. 渲染 /admin 面板前端页面 (反代至 Pages 页面)
return await proxyStaticPages('/admin' + url.search);
```

于是返回的是**管理后台的 HTML 页面**，前端 `response.json()` 解析 HTML 抛错 → 提示「加载日志失败」。
原版存在该接口（`_worker.js` 第 112-114 行 `else if (访问路径 === 'admin/log.json')`），属新版能力缺失。

**修复**：新增 `/admin/log.json`（返回 JSON 数组）+ 全链路日志落盘（登录、改密、保存配置、优选轮换、解析失败等）。

---

### 故障 2：面板选择性配置保存后刷新没保存上 【已复现，三种不同成因】

前端 `admin.html` 实际调用的后端接口与新版 Worker 实现的接口严重不匹配：

| 前端调用 | 新版 Worker 实现 | 实测结果 |
|---|---|---|
| `GET/POST /admin/config.json` | 有 | 无 KV → **500**；有 KV → 正常 |
| `POST /admin/cf.json` | **仅 GET，POST 被忽略** | 返回 `request.cf` 连接信息，**保存无效** |
| `POST /admin/tg.json` | **完全没有** | 返回 HTML 页面（200） |
| `GET /admin/init`（重置） | **完全没有** | 返回 HTML 页面（200） |
| `GET /admin/getCloudflareUsage` | **完全没有** | 返回 HTML 页面（200） |
| `GET /version` | **完全没有** | 被反代到伪装站，返回 Bing 页面 |

**证据 1（无 KV 时保存直接失败）**
```
POST /admin/config.json
  -> 500 {"error":"未绑定 KV 数据库，无法持久化存储配置"}
（对应 _worker.js 第 330-332 行）
```

**证据 2（CF 配置保存被吞掉）**
```
POST /admin/cf.json  body={"a":1}
  -> 200 application/json
     {"httpProtocol":"HTTP/1.1","clientAcceptEncoding":"identity", ... }
```
返回的是 Cloudflare 连接信息（`request.cf`），而不是保存结果 —— 代码第 379-384 行无条件走 GET 分支。

**证据 3（TG/重置等接口静默失败，且前端误判成功）**
```
POST /admin/tg.json -> 200 text/html; charset=utf-8  (返回 admin.html)
GET  /admin/init    -> 200 text/html; charset=utf-8  (返回 admin.html)
```
关键点：这些**返回 200 而非 4xx/5xx**，前端 `if (!response.ok) throw` 判定为成功并弹出「✅ 已保存」，
实际什么都没存 —— 这正对应「保存了但刷新后没有」。

**修复**：补齐 `cf.json` 写入、`tg.json`、`init`、`getCloudflareUsage`、`version`；
未知 `/admin/*` 一律返回 **JSON 404**（不再回吐 HTML）；KV 未绑定时降级为内存存储并明确提示，不再直接 500。

---

### 故障 3：Web 管理面板密码无法修改 【已复现】

**证据**：
1. 全仓搜索 `frontend/admin.html`，「密码」二字仅出现 **1 次**，且是无关内容
   （第 10824 行 `crypto.cloudflare.com（Cloudflare密码学服务）`）→ **面板根本没有改密码入口**；
2. `_worker.js` 全文**没有任何改密码接口**，管理口令只从环境变量读取（第 206 行）：
   ```js
   const adminPassword = (env.ADMIN || env.admin || env.PASSWORD || env.password || env.UUID || DEFAULT_UUID).trim();
   ```
3. 因此即使把密码写进 `config.json`，登录校验也不会读取它 —— 密码在架构上就无法在面板里改。

**修复**：
- 新增 `POST /admin/password`（校验旧密码 → 写入 KV `ADMIN_PASSWORD` → 登录时优先读 KV）；
- 面板注入「修改管理密码」弹窗与右下角入口按钮；
- 按用户选择：**保留默认口令可登录，但未改密前禁止任何写操作**（返回 403 + `needChangePassword`）。

> ⚠️ 实现要点：后台页面默认从第三方 Pages 回源（见第四节），
> **直接修改本地 `frontend/admin.html` 并不会生效**。
> 因此修复版在回源 `/admin` 页面时**动态注入**改密组件（实测生效：页面由 892,950 字节增至 897,909 字节，
> 且含 `pwdChangeModal` / `pwdSaveBtn` / `/admin/password` 调用）。
> 若你把 `frontend/` 自托管并通过 `PAGES_BASE` 指向，可用 `INJECT_PWD_WIDGET=false` 关闭注入。

---

### 故障 4：订阅不可用 / 解析异常 【已复现，附准确定性】

需要说明：在仿真环境中 `/sub` 返回的 base64 **可以被正常解码**，
因此「完全无法解析」的客户端报错文案无法在沙箱内复现；
但实测确认了 3 个会直接导致订阅不可用/不稳定的缺陷：

**证据 1：节点地址每次请求随机变化（6 次请求出现 2 种地址）**
```
第1-4次: vless://...@cloudflare.net:443?...
第5-6次: vless://...@cdn.xn--b6gac.eu.org:443?...
是否恒定: False
```
根因（第 472、476 行）：节点地址取自 `getBestProxyIP(env)`，
无 KV 时退化成从 `DEFAULT_PROXY_IPS` 随机取一个 —— 这些是**出站备用 ProxyIP**，
并非用户自有域名，客户端拿它做 TLS 握手（SNI=worker 域名）大概率失败；
而且每次订阅内容都变，客户端缓存与比对也会异常。
原版做法（第 413-418 行）是从用户自配置的优选节点/HOSTS 取地址。

**证据 2：Clash 订阅把所有流量直连**
```
GET /sub?clash -> YAML 解析成功, rules: ['MATCH,DIRECT']
```
`MATCH,DIRECT` 意味着代理形同虚设（第 508 行）。

**证据 3：格式支持不全** —— 新版只有 vless base64 与 Clash 两种，
原版支持 `mixed / clash / singbox / surge` + subconverter 订阅转换（原版第 458-494 行）。

**修复**：节点地址改用「后台保存的优选节点(ADD.txt) → 否则 Worker 域名(HOSTS/hostname)」，
输出恒定；新增 sing-box 输出；Clash 规则改为 `MATCH,PROXY` 并补齐 `proxy-groups`。

---

## 三、额外发现的问题（非用户报告，但确证存在）

| # | 问题 | 等级 | 证据 | 处置 |
|---|---|---|---|---|
| 5 | **出厂默认口令写死在代码与文档里** | 高 | `_worker.js:20`、`wrangler.toml`、`README.md` 均为 `d342d11e-d424-4583-b36e-524ab1f0afa4`，且仓库公开 | 保留默认值（按用户选择）+ 强制改密 + 登录失败限流 |
| 6 | **KV 未绑定 → 核心功能 500** | 高 | `wrangler.toml` 中 `id = "YOUR_KV_NAMESPACE_ID"` 占位符；实测保存返回 500 | 降级为内存存储 + `storage` 字段明确提示 + 配置文件醒目说明 |
| 7 | **反代伪装回传上游 Cookie** | 中 | 实测 `GET /version` 响应带 `Set-Cookie: MUID=...; domain=.bing.com` —— 把伪装站的 Cookie 种到了本站域名 | 反代响应删除 `Set-Cookie`/`Set-Cookie2`，请求头删除 `cookie` |
| 8 | **错误信息回显内部资源地址** | 低 | 第 424 行 `资源地址: ${targetUrl}` | 去掉回显，改为通用提示 |
| 9 | **鉴权 Cookie 绑定 User-Agent** | 中 | 第 268 行 `md5Hex(userAgent + secretKey + adminPassword)`：UA 变化（换浏览器/升级）即掉线 | 改为 `md5(secretKey + password)`，与 UA 解耦 |
| 10 | **Cookie 硬编码 Secure，本地 http 调试登录失败** | 低 | 第 296 行 `HttpOnly; Secure` | 仅在 https 下加 `Secure` |
| 11 | **登录无节流** | 中 | 无失败计数逻辑，配合公开默认口令可被爆破 | 每 IP 5 分钟内 8 次失败后 429 |
| 12 | **默认配置字段缺失，与前端不匹配** | 中 | 前端读取 `订阅转换配置/TG/CF/SS/ECHConfig/反代/通知/加载时间` 等 16 个字段；`generateDefaultConfig`（第 434-452 行）只给 11 个，且后端字段是 `TIME`、前端读 `加载时间` | 默认配置补齐全部字段并对齐命名 |
| 13 | **BufferPool 首包缓冲复用隐患** | 中 | 第 819-824 行：`send(mergedBuf.subarray(0,totalLen))` 后立刻 `release()` 归还池，缓冲区可能被其它连接复用 | 改为 `slice()` 拷贝后再回收 |
| 14 | **压缩包内 `frontend/*.html` 是死文件** | 低 | `_worker.js` 中 `admin.html`/`login.html`/`noKV`/`noADMIN` 引用数均为 **0**；Workers 部署也不含该目录 | 保留文件并在 README 说明；`noKV`/`noADMIN` 补上路由 |
| 15 | **版本号互相矛盾** | 低 | README 写 2.2、`_worker.js` 头部 2.1、`package.json` 2.1.0、`wrangler.toml` name `edgetunnel-2` | 统一为 2.2.0 |
| 16 | **UDP 未真正支持** | 中 | 第 651-654 行 `if (isUdp && port !== 53) cleanup()`，且全文件无任何 UDP 数据帧封装；原版有 `forwardataudp` | 保留 TCP，报告中标为已知限制（未补齐，避免引入未经充分验证的协议实现） |
| 17 | 客户端粗暴断线时出现未捕获异常 | 低 | 仿真日志出现 `Uncaught Error: Network connection lost.`；优雅关闭/正常断开**不会**出现 | 已加固清理路径；标记为运行时噪声 |

---

## 四、已排除的怀疑项（避免误判）

| 怀疑项 | 结论 | 依据 |
|---|---|---|
| 面板源站 `edt-pages.github.io` 是仿冒/被劫持站点 | **排除** | 响应头 `server: GitHub.com`、`x-github-request-id`、证书为 Let's Encrypt 签发的 `*.github.io`，确为 GitHub Pages；根路径的 nginx 欢迎页只是该仓库的 `index.html` |
| 面板前端依赖第三方 Pages 是新版引入的 | **不是新问题** | 原版 `_worker.js` 第 4 行同样为 `const Pages静态页面 = 'https://edt-pages.github.io'` —— 属继承行为 |
| `Set-Cookie` 在 workerd 下无法设置（导致登录失败） | **排除** | 最小实验：/a(`headers.set`)、/b(`append`)、/c(构造入参)、/d(带 Secure) 四种方式均正常返回 `Set-Cookie` |
| 0-RTT Early Data 失效 | **排除，功能正常** | 裸 TCP 握手带 `Sec-WebSocket-Protocol` 实测：服务端读到 90 字节 early data，解析出 `example.com:80`，返回帧为 `00 00` + `HTTP/1.1 200 OK`（1070 字节） |
| 101 响应前调用 `send()` 会失败 | **排除** | 探针中 `BEFORE-101` 与 `AFTER-101` 两条消息客户端均收到 |

> 说明：中途用 `ws` 库测试 early data 时曾出现「收不到数据」的假象，
> 原因是 `ws` 客户端在服务端回显子协议时抛 `Server sent a subprotocol but none was requested`，
> 属**测试客户端行为**，不是产品缺陷 —— 已用裸握手复核推翻该结论。

---

## 五、原版有而新版缺失的能力（本次已补齐 / 未补齐）

| 能力 | 原版 | 新版 | 本次 |
|---|---|---|---|
| 后台日志 `admin/log.json` | ✅ | ❌ | ✅ 补齐 |
| CF 用量 `admin/getCloudflareUsage` | ✅ | ❌ | ✅ 补齐 |
| 订阅格式 mixed/clash/singbox/surge | ✅ | 仅 2 种 | ✅ 补齐 mixed/clash/sing-box（surge 需 trojan 转换，未实现） |
| 订阅转换后端（subconverter） | ✅ | ❌ | ❌ 未补齐（依赖外部转换服务，需用户自配 SUBAPI） |
| 多协议 SS / trojan | ✅ | ❌ | ❌ 未补齐（超出本次故障修复范围，且需大量协议实现验证） |
| UDP / DoH | ✅（`forwardataudp`、`DoH查询`） | ❌ | ❌ 未补齐（见问题 16） |
| 改密能力 | ❌ | ❌ | ✅ 新增（两边都没有） |
| 0-RTT / 背压 / Cron 优选 | ❌ | ✅ | ✅ 保留并验证 |

---

## 六、修复后回归测试结果（本地 workerd 实测）

| 用例 | 修复前 | 修复后 |
|---|---|---|
| `GET /admin/log.json` | 200 text/html（页面） | **200 application/json**，日志数组，已有 6 条记录 |
| `POST /admin/cf.json` | 200 返回 `request.cf`，未保存 | **200 保存到 KV**，回读一致 |
| `POST /admin/tg.json` | 200 text/html，未保存 | **200 保存**，回读 `{"启用":true,"Token":"t"}` |
| `GET /admin/init` | 200 text/html | **200 JSON**，配置已重置（UUID 回到默认值） |
| `POST /admin/config.json`（无 KV） | 500 | **200**，降级内存并提示 |
| `GET /version` | 反代到 Bing（HTML） | **200 JSON** `{"Version":"2.2.0",...}` |
| `GET /admin/nope` | 200 HTML 页面 | **404 JSON** `{"error":"未知后台接口: /admin/nope"}` |
| 默认口令下写操作 | 可写（不安全） | **403** `needChangePassword` |
| 改密 + 新密码登录 | 不支持 | **改密成功 → 新密码登录成功 → Cookie 下发** |
| 订阅稳定性（3 次） | 地址随机变化 | **3 次内容完全一致** |
| `?clash` 规则 | `MATCH,DIRECT` | **`MATCH,PROXY`** + proxy-groups |
| `?singbox` | 不支持 | **200 JSON**，4 个 outbound |
| 错误 token | 403；无 token 200 | 错误 403；无 token 200（可通过 `REQUIRE_SUB_TOKEN` 强制） |
| 代理转发（普通首包） | 正常 | **正常**（`HTTP/1.1 200 OK`） |
| 代理转发（0-RTT early data） | 正常 | **正常**（1070 字节，`00 00` + HTTP 200） |
| 错误 UUID | 连接关闭 | **连接关闭**（正确拒绝） |
| 语法检查 `node --check` | — | 通过 |

---

## 七、遗留风险与使用建议（务必阅读）

1. **必须绑定 KV**，否则配置只存在内存里，Worker 冷启动后会丢失（界面会提示 `storage: memory`）。
2. **首次登录请立即改密码**：默认口令是公开的，不改则后台任何人可进。
3. **面板前端仍从 Pages 源站拉取**（与官方原版一致）：可用变量 `PAGES_BASE` 指向自建静态托管，
   压缩包 `frontend/` 目录已提供页面文件可供自行托管。
4. ~~未提供 UDP/DoH 与多协议（SS/trojan）能力~~ —— **该条已在 2.3.0 中补齐**，见第八节。
5. 沙箱内无法访问线上 Workers 实例，以上结论来自源码 + 本地 workerd 仿真；
   线上表现受账户配置、绑定与网络环境影响，部署后请以 `/admin/status` 返回为准。


---

## 八、2.3.0 能力补齐（本节全部附可复现证据）

### 8.1 多协议：Trojan over WebSocket（ADD-01）

**规范依据**：trojan 请求头为 `hex(SHA-224(password))`（56 字节）+ `CRLF` + `cmd`(1B) +
`ATYP` + `ADDR` + `PORT`(2B) + `CRLF` + payload；`cmd=0x01` 为 TCP、`0x03` 为 UDP。

**实现难点与处理**：Cloudflare Workers 的 `crypto.subtle` **不支持 SHA-224**，
因此用纯 JS 实现 SHA-224（SHA-256 压缩函数 + SHA-224 初始向量 + 截断至 28 字节）。

**证据（tests/sha224.test.mjs）**：与 Node 内置 `crypto.createHash('sha224')` 逐例比对：

```
SHA-224 与 Node 内置实现比对：137 个用例，通过 137，失败 0
```

用例覆盖 0~130 字节的全部填充边界 + 中文 + emoji。

> **过程中发现并修复的真实缺陷**：首版填充长度按 `len+9` 计算，
> 导致口令长度 ≡ 55 (mod 64) 时多算一个块、哈希值错误、合法客户端被拒。
> 修正为 `len+8` 后 137 例全通过。该缺陷若不测填充边界根本发现不了。

**握手证据（tests/protocol.test.mjs）**：

```
=== B. Trojan over WS ===
  [PASS] Trojan 握手+转发 字节=1064
  [PASS] Trojan 错误口令被拒 收到字节=0
```

### 8.2 多协议：Shadowsocks AEAD over WebSocket（ADD-02）

**为什么是 SIP007 而不是 SIP022**：SIP022（Shadowsocks 2022）规范明确要求
`session_subkey := blake3::derive_key(context: "shadowsocks 2022 session subkey", key_material: key + salt)`，
而 Workers 无 BLAKE3 原生实现。纯 JS 重写 BLAKE3 后**无法与真实客户端做互操作验证**，
因此不提供，避免出现「看起来支持、实际连不上」的功能。

本次实现的是 **SIP007（Shadowsocks AEAD）**，也正是 v2ray-plugin 走 WebSocket 时实际使用的格式：

- 主密钥：`EVP_BytesToKey(MD5)`，长度 16（aes-128-gcm）/ 32（aes-256-gcm）
- 会话子密钥：`HKDF-SHA1(salt=salt, info="ss-subkey")`
- nonce：12 字节小端计数器，从 0 开始，每块 +1
- 分块：salt → [2B 长度密文块] → [载荷密文块] …

**证据**：

```
=== C. Shadowsocks AEAD over WS (SIP007) ===
  [PASS] SS aes-128-gcm 解密并转发 收到密文字节=1114 明文字节=1064
  [PASS] SS 错误口令被拒 收到字节=0
```

（aes-256-gcm 用 `SS_METHOD=aes-256-gcm` 的实例单独跑，同样 7/7 通过。）

测试是**双向闭环**：客户端按 SIP007 加密 → Worker 解密出目标地址并转发 →
回程数据由 Worker 加密 → 测试脚本解密后比对，明文字节数与直连一致（1064 字节）。

**为什么 SS 用独立路径 `/ss`**：SS 报文是纯密文、没有特征头，无法在同一个 WS 路径上
与 VLESS/Trojan 自动区分，因此单独走 `SS_PATH`（默认 `/ss`）。

### 8.3 UDP：DNS over HTTPS（ADD-03）

**平台限制**：Cloudflare Workers 没有 UDP socket，只能发出 `fetch` 请求。
因此实现边界是**明确的**：

| 目标端口 | 行为 |
|---|---|
| `53` | UDP 报文按 2 字节长度分包 → 取出 DNS 查询 → POST 到 DoH 上游 → 应答封回 `2字节长度+数据` |
| 其它 | 明确拒绝并写入日志（不是静默丢弃） |

**证据**：

```
=== D. UDP：DNS over HTTPS ===
  [PASS] UDP DNS 查询经 DoH 返回应答 字节=63 长度字段=61 响应体=61 QR=true answers=2
  [PASS] 非 53 端口 UDP 被拒绝（无回包） 收到字节=0
```

`QR=true`、`answers=2` 说明返回的是真实 DNS 应答报文，不是占位数据。
DoH 上游可换（`DOH_URL`），自检接口 `/admin/doh-test` 实测 `latency=18ms`。

### 8.4 subconverter 订阅转换（ADD-04）

内置 7 种输出：`clash / singbox / surge / quanx / loon / mixed / raw`，
按 UA 自动识别（Clash / Mihomo / sing-box / Surge / Quantumult / Loon）。
配置了 `SUBAPI` 时可转发外部 subconverter，**失败自动回退内置**，不会让订阅挂掉。

**证据（tests/subscription.test.py）**：

```
  [PASS] Clash 含 vless/trojan/ss          types=['vless', 'trojan', 'ss']
  [PASS] Clash 规则走代理                   rules=['MATCH,PROXY']
  [PASS] Clash 的 ss 节点带 v2ray-plugin
  [PASS] sing-box 含三种协议                types=['vless', 'trojan', 'shadowsocks']
  [PASS] Surge 输出含三种协议
  [PASS] Quantumult X 输出含三种协议
  [PASS] Loon 输出含三种协议
===== 订阅与接口测试：通过 22，失败 0 =====
```

**诚实标注**：Clash（YAML 解析 + 字段校验）与 sing-box（JSON 解析 + 字段校验）做了结构化验证；
**Surge / Quantumult X / Loon 是按公开文档语法生成的，未做真机导入验证**。
若某客户端导入异常，请配置 `SUBAPI` 走成熟 subconverter。

### 8.5 KV 名称写死与诊断（ADD-05）

**写死的固定值**（`wrangler.toml` + `setup-kv.mjs`）：

| 项 | 固定值 |
|---|---|
| 命名空间名称（title） | `edgetunnel-kv` |
| 绑定名（binding） | `KV` |

`node setup-kv.mjs` 会自动创建（或复用已有同名命名空间）并把 id 回填到 `wrangler.toml`；
也支持 `node setup-kv.mjs <ID>` 手动写入、`--check` 只做检查。脚本已实测 5 种输入路径均符合预期。

另外新增 `/admin/kv` 诊断接口，直接回答「到底连没连上」：

```json
{ "bound": true, "binding": "KV", "storage": "kv", "write": true, "read": true, "healthy": true }
```

未绑定时，面板顶部会显示橙色告警条（说明要创建 `edgetunnel-kv`、binding 填 `KV`）。

**回归证据**：2.2.0 的 54 项回归在 2.3.0 代码上仍然全绿（带 KV / 不带 KV 两种场景）：

```
===== 汇总：通过 54 项，失败 0 项 =====
```

---

## 九、2.3.0 测试总汇

| 测试 | 命令 | 结果 |
|---|---|---|
| SHA-224 正确性 | `node tests/sha224.test.mjs` | 137 / 137 |
| 协议互操作（VLESS/Trojan/SS/UDP） | `PORT=8792 node tests/protocol.test.mjs` | 7 / 7（aes-128 与 aes-256 实例各一轮） |
| 订阅转换 + 新增接口 | `PORT=8794 python3 tests/subscription.test.py` | 22 / 22 |
| 2.2.0 全量回归（两种场景） | `python3 tests/regression.py` | 54 / 54 |
| 语法检查 | `node --check _worker.js` | 通过 |

---

## 十、2.3.0 已知限制（不掩饰）

1. **Shadowsocks 2022（SIP022）未实现** —— 强制 BLAKE3，无原生支持且无法验证互操作性（详见 8.2）。
2. **Surge / Quantumult X / Loon** 按文档语法生成，**未做真机导入验证**；Clash 与 sing-box 已结构化验证。
3. **UDP 仅 53 端口** —— 平台无 UDP socket，属实现边界而非缺陷。
4. 面板前端仍默认从第三方 Pages 回源（与原版一致）；需完全自控请设置 `PAGES_BASE` 自托管。
5. 全部结论来自源码审计 + 本地 workerd 仿真；沙箱内无法访问线上实例，
   线上表现请以部署后 `/admin/status`、`/admin/kv` 的返回为准。
