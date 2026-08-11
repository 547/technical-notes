# Mac + cloudflared 多端口隧道 + launchd 常驻 配置留档

> 适用场景：本地服务（开发/自托管）通过 Cloudflare 隧道暴露到固定公网域名，Mac 重启或进程被杀后自动恢复。
> 本文基于 2026-08-10 在 momo 的 Mac mini（Apple Silicon, macOS 26.4.1）上的真实配置整理。

---

## 一、当前已落地状态

| 项 | 值 |
|---|---|
| cloudflared 版本 | 2026.7.3（Homebrew 安装） |
| 隧道名 / ID | `my-tunnel` / `58960ddb-a163-4e4f-a7ba-86831cd0dd8f` |
| 域名 | `54715471.xyz`（NameSilo 注册，2032-08-12 到期；NS 已在 Cloudflare） |
| codebuddy 子域 | `codebuddy.54715471.xyz` → 本地 `localhost:50000`(2026-08-11 由 59395 改为 50000) |
| jenkins 子域 | `jenkins.54715471.xyz` → 本地 `localhost:8080` |
| 常驻方式 | launchd（用户级，免 sudo），开机自启 + 进程保活 |
| 公网保护 | 未加 Cloudflare Access（两服务自身有登录密码，按需可补） |

**访问效果**
- `https://jenkins.54715471.xyz` → 正常进入 Jenkins 登录页 ✅
- `https://codebuddy.54715471.xyz` → 页面能进，但内部「连接失败」（见已知问题）

---

## 二、架构

```
公网用户
   │  https
   ▼
Cloudflare 边缘（DNS: codebuddy/jenkins.54715471.xyz → 隧道）
   │  QUIC/HTTP2 加密隧道
   ▼
本机 cloudflared (launchd 托管, 读 config.yml)
   │  按 hostname 分流 (ingress)
   ├─► http://localhost:59395   (codebuddy)
   └─► http://localhost:8080    (jenkins, brew jenkins-lts)
```

---

## 三、关键文件清单

| 文件 | 作用 |
|---|---|
| `~/.cloudflared/config.yml` | 隧道 ingress 规则（子域→本地端口映射） |
| `~/.cloudflared/cert.pem` | Cloudflare 账号授权凭证（已登录过，勿删） |
| `~/.cloudflared/58960ddb-....json` | 隧道 credentials 文件 |
| `~/Library/LaunchAgents/com.cloudflare.cloudflared.plist` | launchd 自启任务定义 |

---

## 四、config.yml 内容（实际生效）

```yaml
tunnel: 58960ddb-a163-4e4f-a7ba-86831cd0dd8f
credentials-file: /Users/momo/.cloudflared/58960ddb-a163-4e4f-a7ba-86831cd0dd8f.json
protocol: http2          # 关键：绕开被中间网络掐断的 QUIC，走 TCP 443（见第十二节）

ingress:
  - hostname: codebuddy.54715471.xyz
    service: http://localhost:50000
  - hostname: jenkins.54715471.xyz
    service: http://localhost:8080
  - service: http_status:404
```

> ⚠️ 最后一条 `http_status:404` 是兜底规则，**必须放最后**，否则匹配顺序会出问题。
> 🔴 **必须加 `protocol: http2`**：本机出口网络会中途掐断 QUIC（cloudflared 默认协议），导致隧道连接timeout、Cloudflare 后台显示离线。强制走 TCP 443 的 HTTP/2 可根治（详见第十二节）。

---

## 五、launchd 自启 plist（实际生效）

### 5.1 cloudflared 隧道

路径：`~/Library/LaunchAgents/com.cloudflare.cloudflared.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.cloudflare.cloudflared</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/cloudflared</string>
        <string>--config</string>
        <string>/Users/momo/.cloudflared/config.yml</string>
        <string>tunnel</string>
        <string>run</string>
        <string>my-tunnel</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/cloudflared-launchd.out.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/cloudflared-launchd.err.log</string>
    <key>WorkingDirectory</key>
    <string>/Users/momo</string>
</dict>
</plist>
```

> 🔴 **踩坑记录**：`--config` 是全局参数，必须放在 `tunnel` 子命令**之前**。
> 写成 `tunnel run --config ...` 会导致 cloudflared 解析失败、打印 help 后退出的假死。
> 正确顺序：`cloudflared --config <file> tunnel run <name>`。

### 5.2 codebuddy 服务（端口 50000，2026-08-11 加入开机自启）

路径：`~/Library/LaunchAgents/com.codebuddy.server.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.codebuddy.server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/@tencent-ai/codebuddy-code/bin/codebuddy</string>
        <string>--serve</string>
        <string>--host</string>
        <string>127.0.0.1</string>
        <string>--port</string>
        <string>50000</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>WorkingDirectory</key>
    <string>/Users/momo</string>
    <key>StandardOutPath</key>
    <string>/tmp/codebuddy-launchd.out.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/codebuddy-launchd.err.log</string>
</dict>
</plist>
```

> 🔴 **踩坑记录（codebuddy）**：launchd 环境**没有 PATH**，`codebuddy` 是 `#!/usr/bin/env node` 脚本，直接填 `/usr/local/bin/codebuddy` 会因 `env: node: No such file or directory` 启动失败。
> 修法：ProgramArguments 第一项写**绝对路径的 node**（`/usr/local/bin/node`，v22.22.2），第二项写 codebuddy 真实脚本绝对路径 `/usr/local/lib/node_modules/@tencent-ai/codebuddy-code/bin/codebuddy`，**不要依赖 `env node`**。
> 本地起法（手动无托管时）：`codebuddy --serve --host 127.0.0.1 --port 50000`

---

## 六、从零复现步骤（换机器/重装参考）

### 1. 安装 cloudflared
```bash
brew install cloudflared
cloudflared --version
```

### 2. 登录 Cloudflare（授权域名所在账号）
```bash
cloudflared tunnel login
# 浏览器弹出，勾选 54715471.xyz 这个 zone 授权
```

### 3. 建隧道
```bash
cloudflared tunnel create my-tunnel
# 记下输出的 tunnel-id
```

### 4. 写 config.yml（见第四节）

### 5. 把子域名指向隧道（Cloudflare 自动加 CNAME）
```bash
# codebuddy 本地起法（手动，无 launchd 托管）：codebuddy --serve --host 127.0.0.1 --port 50000
#   监听 127.0.0.1:50000；隧道 config.yml 里 codebuddy.54715471.xyz → localhost:50000
cloudflared tunnel route dns my-tunnel codebuddy.54715471.xyz
cloudflared tunnel route dns my-tunnel jenkins.54715471.xyz
```

### 6. 配 launchd 自启（见第五节），然后：
```bash
launchctl load ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
```

### 7. 验证
```bash
curl -sI https://jenkins.54715471.xyz   # 应返回 HTTP 头
curl -sI https://codebuddy.54715471.xyz
```

### 8. ⚠️ Cloudflare SSL 加密模式必须设 Full（否则 codebuddy WebSocket 失败）
- 后台 `SSL/TLS` → 加密模式设为 **`Full`**（不要 Flexible）。隧道回源是 HTTPS，Flexible 会让 Cloudflare→源站走明文，导致 codebuddy 页面内 WebSocket「连接失败」（HTTP 页面能进，但实时通道建不起来）。
- 此设置对 `54715471.xyz` 已做；**eu.org（`wangqi5471.eu.org`）审核通过后同样要设 Full**，否则会复现同样问题。

---

## 七、日常运维命令

```bash
# 看隧道是否在跑
launchctl list | grep cloudflared
pgrep -fl "bin/cloudflared --config"

# 看连接日志
tail -f /tmp/cloudflared-launchd.out.log

# 停掉（launchd 会因为 KeepAlive 立刻拉起，所以这只是临时重启用）
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist

# 重新加载（改了 config.yml 后）
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl load   ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist

# 临时后台跑（不用 launchd，调试用）
nohup cloudflared tunnel run my-tunnel > /tmp/cloudflared-my-tunnel.log 2>&1 &
```

---

## 八、已知问题（已解决）

### ✅ codebuddy 进页面后「连接失败」→ 已解决
- **现象（原）**：`https://codebuddy.54715471.xyz` 能打开登录页并进入，但应用内提示连接失败、重试无效。
- **根因（实锤，2026-08-10 用户解决）**：Cloudflare 该域名的 **SSL/TLS 加密模式** 不是 `Full`。隧道回源是 HTTPS，若加密模式为 `Flexible`（Cloudflare→源站走明文 HTTP），源站返回的安全头/TLS 协商与前端 WS 升级冲突，导致 WebSocket 连接建立失败；HTTP 普通页面仍能通（所以能进页面）。
- **解决**：Cloudflare 后台 → 该域名（或 zone 级）`SSL/TLS` → 加密模式设为 **`Full`**（需 Cloudflare 与源站均用 TLS，与隧道回源匹配）。改完 codebuddy 页面内 WebSocket 立刻恢复正常，「连接失败」消失。
- **对比验证**：Jenkins 同样走隧道但正常，是因为 Jenkins 回源对加密模式不敏感；codebuddy 的实时 WS 通道对加密模式敏感，必须 `Full`。
- **结论**：隧道本身 + config.yml 无需改动，问题在 Cloudflare 边缘的加密模式配置，不在客户端。

---

## 九、可选的加固（未做，按需）

- **加 Cloudflare Access 登录保护**：Zero Trust 台 `one.dash.cloudflare.com` → 访问控制 → 应用程序 → 添加 Self-hosted 应用，Policy 设 `Allow + 你的邮箱`。适合服务自身无密码时。
- **系统级自启**：若要多用户/开机早于登录，可放 `/Library/LaunchDaemons/`（需 sudo）。当前用户级已满足需求。

---

## 十二、故障排查：隧道离线（QUIC 被掐 → 改用 http2）

> 实战时间：2026-08-11。现象：昨天配通正常，今早发现 Cloudflare 后台 tunnel 显示离线。

### 12.1 现象
- Cloudflare 后台 tunnel 状态 = `Offline` / 离线
- 公网访问返回 `HTTP 530`（边缘连不上隧道）
- 进程其实活着（`pgrep` 能看到），launchd 不拉起（KeepAlive 只管进程死活，不管连接健康）
- 日志狂刷：`timeout: no recent network activity` / `control stream encountered a failure`

### 12.2 根因
- cloudflared 默认优先 **QUIC（UDP 443）**。本机出口网络（运营商/跨境出口/防火墙）**会中途掐断或限速 QUIC 数据流**，握手成功但传输 timeout。
- 关键证据：手动前台跑时 CONNECTIVITY PRE-CHECKS 显示 `UDP Connectivity PASS`，但随后仍 `Failed to dial ... timeout`。说明底层 UDP 通、但 QUIC 长连接被掐。
- **为什么昨天正常今天坏**：同一配置、同一 Mac，QUIC 在 8/10 17:01 还通，10:26 就被掐。是中间网络策略中途变了脸（运营商对 QUIC 的间歇性管控），不是配置问题。

### 12.3 修法（已固化到 config.yml）
在 `config.yml` 顶部加一行，强制 http2：
```yaml
protocol: http2
```
然后重载：
```bash
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl load   ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
```
恢复标志：日志出现 `SUMMARY: Environment is healthy. cloudflared will use 'http2' as primary protocol.`，公网回到 `HTTP/2 403`（服务活着）。

### 12.4 临时手动重启（网络抖动卡死时）
若隧道卡在重试循环、launchd 不拉起，手动强制重启：
```bash
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
pkill -9 -f "bin/cloudflared --config"
launchctl load   ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
```

### 12.5 一键诊断命令
```bash
# 进程在不在
pgrep -fl "bin/cloudflared --config"
# 实时日志看是 QUIC 还是连接问题
tail -f /tmp/cloudflared-launchd.err.log | grep -iE "ERR|protocol|Registered|healthy"
# 公网通不通
curl -sI https://jenkins.54715471.xyz | head -1
# 期望：HTTP/2 403（正常） 或 HTTP/2 530（隧道离线）
```

### 12.6 后续隐患
- 若某天 http2 也偶发离线，可能是更上层网络抽风。可考虑加连通性监测脚本配合 launchd 自动重启（暂未做，按需再加）。
- eu.org 申请的 `wangqi5471.eu.org` 审核通过、且确认出口稳定后，可照第十一节扩展更多子域。

---

## 十、快速结论

这套配置已实现：固定域名 + 多端口分流 + Mac 重启/进程崩溃自动恢复。已固化 `protocol: http2` 绕开 QUIC 被掐问题（见第十二节）。codebuddy 也已加入开机自启 (`com.codebuddy.server`, 端口 50000, KeepAlive 生效)，其页面内 WebSocket 连接失败问题已通过 Cloudflare SSL 加密模式改 `Full` 解决（见第八节）。所有已知问题均已闭环，Jenkins 完全正常。

---

## 十一、免费域名申请（eu.org）—— 2026-08-10~11 实战记录

> 目的：给隧道加一个永久免费、可改 NS 到 Cloudflare 的备用固定域名。
> 结论：已申请 `wangqi5471.eu.org`，工单号 `20260811024453-arf-36049`，待审核。

### 11.1 哪些「免费域名」能接 Cloudflare 隧道

命名隧道（`cloudflared tunnel`）要求域名的 **NS 必须能改到 Cloudflare**。因此：

| 域名来源 | 能否改 NS 到 Cloudflare | 结论 |
|---|---|---|
| eu.org（真实二级域） | ✅ | 推荐，永久免费 |
| us.kg / lu.kg / 各种 .kg | ❌（NS 锁定在 website.org） | 不能接隧道 |
| duckdns / no-ip 等动态 DNS | ❌ | 只能玩快速隧道 |
| Freenom .tk/.ml/.ga/.cf/.gq | ❌（已停发/不可靠） | 别用 |

### 11.2 踩坑：website.org（lu.kg）是坑

- 在 `https://website.org/` 随便选了个 `547.lu.kg`，实测 NS 被锁死在 `ns1.website.org.`，**改不了 NS**，无法接 Cloudflare 隧道。
- `nic.kg` / `us.kg` 注册入口现已失效（打开跳 `website.org/404`），不可靠，别耗时间。
- **结论：website.org / lu.kg 这类只能当玩具页，不能用于隧道。**

### 11.3 eu.org 申请流程（实际可用）

**Step 1：注册账号**
- 打开 `https://nic.eu.org/`，点 **Register**
- ⚠️ 表单的 **Name 字段是「登录用户名」，不是真实姓名**，只接受 ASCII（小写字母+数字）
- 填中文 `王琦` 会报 `Enter a valid value.` → 改成 `wangqi` 即可

**Step 2：申请域名（Request a domain / New domain）**
- **Complete domain name** 字段填**完整域名**（前缀+后缀），如 `wangqi5471.eu.org`
- 系统先查重：返回 `Domain not found` = 「没人占用 = 可用」✅（不是报错）；返回已存在才是真被占
- `momo` / `wangqi` 这类短词已被占，需用组合（`wangqi5471`、`momo-home` 等）

**Step 3：填 Nameservers（关键）**
- 表单要 Cloudflare 的两条 NS，**IP 栏留空**（Cloudflare 的 NS 在 `cloudflare.com` 下，是外部域名，不需要 glue 记录）
- 提交前必须先在 Cloudflare 后台 **Add a Site 添加 `wangqi5471.eu.org`**，拿到那两条 NS
- 本例填的是 `isaac.ns.cloudflare.com` / `lindsey.ns.cloudflare.com`（注意：与 `54715471.xyz` 复用同一对 NS 主机名是正常现象，前提是新 zone 已建）

**Step 4：提交与检查**
- 点 Submit 后系统自动跑 NS/SOA 检查：连 Cloudflare 两条 NS，SOA、NS 记录均 `ok`，无报错
- 最后输出：
  ```
  No error, storing for validation...
  Saved as request 20260811024453-arf-36049
  Done
  ```
- **工单号 `20260811024453-arf-36049` 记好备查**

**Step 5：等审核**
- eu.org 是志愿者运营，审核 **几天到两三周**，留意注册邮箱
- 审核期间**不要改 NS、不要删 Cloudflare 里的 `wangqi5471.eu.org` 站点**（保持 SOA/NS 可查，否则审核失败）

### 11.4 审核通过后的落地步骤（备着）

1. 邮箱收到批准通知，回 Cloudflare 确认 `wangqi5471.eu.org` 状态变 `Active`
2. 绑定子域到隧道：
   ```bash
   cloudflared tunnel route dns my-tunnel codebuddy.wangqi5471.eu.org
   cloudflared tunnel route dns my-tunnel jenkins.wangqi5471.eu.org
   ```
3. 在 `~/.cloudflared/config.yml` 的 `ingress` 加两条：
   ```yaml
   - hostname: codebuddy.wangqi5471.eu.org
     service: http://localhost:50000
   - hostname: jenkins.wangqi5471.eu.org
     service: http://localhost:8080
   ```
4. 改完重载 launchd：
   ```bash
   launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
   launchctl load   ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
   ```

### 11.5 一句话总结

免费域名要接隧道 → 认准 **eu.org**（NS 可改到 Cloudflare）；website.org/lu.kg/us.kg 这类 NS 锁死的都是坑。申请时 Name 填 ASCII 用户名、域名填完整、NS 填 Cloudflare 两条且 IP 留空、提交后只等审核。
