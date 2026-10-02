# EQ旗舰生产部署版 (含 Web 管理面板与 2.2 高性能内核)

基于 Cloudflare Workers 架构构建的新一代 VLESS over WebSocket 代理服务端。

## ✨ 核心特性

- **完整 Web 后台支持**：修复了精简版无法访问管理后台的问题，原生集成 `/login`、`/admin` 与 `/sub` 订阅管理面板；
- **智能抗主动探测 (Anti-Probing)**：仅对管理与授权路由放行，所有非握手探测和未知路径透明无感反向代理至正常合法源站（绝不返回 400/500 报错）；
- **动态 ProxyIP 自动轮换 (Cron Failover)**：每 5 分钟定时并发探测候选 ProxyIP 的 TCP 握手延迟，最优节点自动写入 KV；出站直连阻断时毫秒级无缝降级；
- **全双工背压流控 (Backpressure Control)**：基于 `desiredSize` 与 `bufferedAmount` 动态协同控流，防止 Worker 内存积压被 CF 强制 Kill；
- **TypedArray 内存池优化 (Buffer Pool)**：消除海量小包处理过程中的重复内存分配，降低 V8 垃圾回收 (GC) 抖动与延迟；
- **0-RTT Early Data 握手加速**：提取 `Sec-WebSocket-Protocol` 首包，节省 1 个 RTT 往返；
- **双向 KV 兼容**：同时兼容 `env.KV` 与 `env.PROXYIP_KV`，老版本一键无缝替换升级。

## 🚀 部署与访问入口

1. **部署方法**：
   - **方式一（Web 在线部署）**：直接复制 `_worker.js` 代码粘贴到 Cloudflare Workers 在线编辑器中保存并部署。
   - **方式二（Wrangler CLI）**：修改 `wrangler.toml` 中的 KV 命名空间 ID，终端执行 `npx wrangler deploy`。

2. **Web 管理后台访问**：
   - 登录地址：`https://你的Worker域名/login`
   - 管理后台：`https://你的Worker域名/admin`
   - 默认密码：UUID默认密码：d342d11e-d424-4583-b36e-524ab1f0afa4
   - 快速订阅：`https://你的Worker域名/<KEY>`

3. **客户端节点订阅**：
   - 订阅通用地址：`https://你的Worker域名/sub`
   - Clash 订阅：`https://你的Worker域名/sub?clash`
