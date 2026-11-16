# iOS ChatGPT App 走旁路由报「地区不可用」的一个解法

> 2026-11-15 实测记录 · 环境：iPhone 18.x / Debian 12 旁路由（xray TPROXY）/ 美国 VPS
> 结论先行：**让 AI 域名解析到"假地址"，靠 xray 的域名嗅探按域名转发**，App 就能进
> 反直觉的地方：给它**正确**的 Cloudflare IP 反而失败
> 文中 IP 已做脱敏（203.0.113.7 是文档保留地址）

---

## 一、症状

| 设备 / 路径 | ChatGPT App | 说明 |
|---|---|---|
| 安卓 + Wi-Fi（走旁路由） | ✅ | 同一出口 |
| iPhone + Wi-Fi（走旁路由） | ❌ 「ChatGPT 在您所在的地区不可用」 | |
| iPhone + 开 Shadowrocket | ✅ | **同一个出口 IP** |
| iPhone 的 Safari / 网页版 | ✅ | 只有 App 不行 |

三组对照说明了两件事：

1. **网络层没问题**——同一台旁路由、同一个出口、同一条代理，安卓能过
2. **IP 也没问题**——同一台 iPhone 换个客户端就能过

所以问题在 **iOS App 自己的判定**

---

## 二、排查过的（都附实测，免得你重走）

| 假设 | 实测 | 结论 |
|---|---|---|
| 出口 IP 被 OpenAI 标记 | 开梯子和走旁路由，`test-ipv6.com` 显示的 IPv4 完全一样（`203.0.113.7`） | ❌ 排除 |
| IPv6 半死导致卡顿 | `test-ipv6.com` → `No IPv6 address detected` | ❌ 排除 |
| WebRTC STUN 泄漏真实 IP | 把手机全部转发 UDP 拒掉，再全端口盯 3 分钟：直连外网记录 **0 条** | ❌ 排除 |
| 加密 DNS 绕过旁路由解析 | xray 里 block 853 / DoH 域名后，日志变成 `-> block` | ✅ 已修（但不足以解决） |
| 设备认证令牌（App Attest / DeviceCheck） | 走网关时 `17.33.0.0/16` 计数 **0**；开代理时日志里有 `register.appattest.apple.com`（224ms，成功） | ⚠️ 认证发生在**登录之后**，卡点在它之前 |
| 分流线路（vmess vs vless） | 两种都试过 | ❌ 无差别 |
| UDP / QUIC 被拒 | 把 UDP 443 接进隧道再试 | ❌ 无差别 |
| MITM 抓包看响应 | OpenAI 官方文档明说：**apps implement certificate pinning** | ❌ 抓不了 |
| 改客户端 / 破解 | 判定在服务端 | ❌ 无意义 |

**一个关键情报**（来自 [V2EX 同症状帖](https://global.v2ex.co/t/1089558)）：

```
正常返回：{"cf_details":"Request is not allowed. Please try again later.", "type":"dc"}
异常返回：{"cf_details":"Something went wrong. You may be connected to a disallowed ISP..."}
```

也就是说 `type":"dc"` 是**正常**的，不是封禁证据（我一开始误判了，绕了很久）

---

## 三、解法

### 3.1 改 DNS：让 AI 域名解析到"假地址"

`/etc/dnsmasq.conf`：

```ini
# AI 域名用国内 DNS 解析 —— 故意拿到污染地址
server=/openai.com/223.5.5.5
server=/chatgpt.com/223.5.5.5
server=/oaistatic.com/223.5.5.5
server=/oaiusercontent.com/223.5.5.5
```

解析结果会像这样（假地址，每次可能不同）：

```
chatgpt.com         → 157.240.1.50     ← 脸书的段（污染）
ios.chat.openai.com → 104.244.46.185   ← 推特的段（污染）
```

### 3.2 前提：xray 必须开嗅探

```json
{
  "tag": "tproxy-in",
  "protocol": "dokodemo-door",
  "settings": { "network": "tcp", "followRedirect": true },
  "streamSettings": { "sockopt": { "tproxy": "tproxy" } },
  "sniffing": {
    "enabled": true,
    "destOverride": ["http", "tls"]
  }
}
```

### 3.3 前提：拦掉手机的加密 DNS

否则手机会绕过旁路由自己用 8.8.8.8 做 DoT/DoH，上面的解析根本不生效。

```json
// xray routing.rules 最前面
{ "type": "field", "port": "853", "outboundTag": "block" },
{ "type": "field", "outboundTag": "block",
  "domain": ["domain:dns.google", "domain:cloudflare-dns.com",
             "domain:one.one.one.one", "domain:dns.quad9.net"] }
```

### 3.4 为什么这样反而能通

```
iPhone 连的是假地址（157.240.1.50）
  → 被旁路由的 TPROXY 截下
  → xray 从 TLS 握手读出真域名（嗅探 destOverride: tls）
  → 按域名转发给上游
  → 由上游重新解析、连到真正可用的边缘
```

**那个假 IP 只是个占位符，从来没被真正连过。** 实测证据：给了假地址之后 App 照样能连上——说明真正建连用的是域名，不是 IP

而之前用干净解析拿到"正确"的 Cloudflare IP，反而落在会回 `403 {"type":"dc"}` 的那个边缘上

---

## 四、验证

```sh
# 1) 解析应该是假地址
getent hosts chatgpt.com            # 例：157.240.1.50

# 2) 旁路由自己走代理要通
curl -x http://127.0.0.1:10809 -o /dev/null -w '%{http_code}\n' https://chatgpt.com/
# 期望 200

# 3) 手机：关掉梯子、只开 Wi-Fi，杀掉 App 重开
```

我这边的结果：连开两次都能进，能发消息

---

## 五、边界（重要）

**这套只对「前面是 Cloudflare 的服务」有效。**

拿 Claude 试过，全部无效：

| 试法 | 结果 |
|---|---|
| 给 `claude.ai` 造个假地址 | ❌ |
| `anthropic` 改走 vless 出站 | ❌ |
| UDP / QUIC 透传 | ❌ |

原因（推断）：`chatgpt.com` 前面是 Cloudflare，域名转发正好落在它的代理层，判定会变；`claude.ai` 是 Anthropic 自己的 IP（`160.79.104.10`），没有那层可借，怎么转都是同一个判定。

**补充（2026-11-16 复查）**：到第二天，Claude 在**所有平台**走旁路由都不可用了（安卓、iOS 一样），
但同一出口 IP 下开客户端（Shadowrocket，用的就是同一个 vless 节点、同一组参数）仍然能过
我们这边配置一个字没改，所以更可能是 Anthropic 侧收紧了判定

**如果 Claude 也要用**：只能手机跑客户端，或者换住宅 / 商宽落地（IP 类型 `ISP`/`Residential`，欺诈分 < 15）

---

## 六、副作用与风险

- 只影响 `/etc/dnsmasq.conf` 里手工指定的那几个域名，其它一律不变
- 这几个域名在国内**本来就需要代理**才能访问，所以"假地址导致直连失败"实际上等于零
- 别把 AI 域名的解析改回"干净"的——那就是失败状态
- 这套依赖嗅探；如果关掉 `destOverride`，假地址就真的连不上了

---

## 七、环境（复现参考）

```
手机：192.168.1.50，网关指向旁路由
旁路由：Debian 12，xray（TPROXY 12345），dnsmasq（53），ipset 存中国 IP 段
上游：美国 VPS，vless(ws+tls, 8443) / vmess(443) / hysteria2(8444)
分流：中国 IP 与 geosite:cn 直连；其它走代理
```

参考：

- [V2EX：iOS ChatGPT App 提示 disallowed ISP](https://global.v2ex.co/t/1089558)
- [OpenAI Help: Corporate network controls（证书绑定说明）](https://help.openai.com/en/articles/20001323-corporate-network-controls-in-chatgpt-enterprise)
- [iOS ChatGPT/Claude 分流与原生商宽](https://jichangzhuanxian.blog/devices/ios-chatgpt-claude-routing)

---

## 八、同一套旁路由上的另一类坑：Twitter 图床被污染（2026-11-17）

现象：X（Twitter）App 里**文字全正常、图片全是灰框**，重开 App 能好一阵又复发

查下来跟上一节是同一类问题的**反面**：

| 项目 | ChatGPT（上一节） | Twitter 图床（这一节） |
|---|---|---|
| 假地址哪来的 | **故意的**（`223.5.5.5` 返回污染结果） | **非故意**（GFW 直接注入） |
| 假地址长什么样 | `157.240.x` / `104.244.x` | `172.64.150.129` / `104.18.37.127`（Cloudflare），而 `pbs.twimg.com` 真身在 Fastly |
| 为什么变成故障 | 假地址只是占位符，从来不被真连 | `domainStrategy: IPIfNonMatch` 会**先本地解析再连**，于是真去连了假 IP |
| 表现 | 恒失败 | **时好时坏**：这次解析到 Fastly 就出图，解析到假 IP 就灰 |

一眼认出污染：

```sh
dig +short A pbs.twimg.com @<旁路由>
# 假：pbs.twimg.com.cdn.cloudflare.net. 172.64.150.129 104.18.37.127   ← pbs 不在 Cloudflare
# 真：dualstack.twimg.twitter.map.fastly.net. 146.75.112.159
```

而且**同一个域名连问几次，答案会在真假之间跳**——这就是"图片时有时无"的来源。

### 8.1 改法（两条，缺一不可）

**① 图床域名走国内干净解析**（`/etc/dnsmasq.conf`）：

```ini
server=/twimg.com/223.5.5.5
server=/twitter.com/223.5.5.5
server=/twitpic.com/223.5.5.5
```

**② 别让 xray 在本地解析**（关键的一条）：

```json
"routing": { "domainStrategy": "AsIs" }     // 原来是 IPIfNonMatch
```

`IPIfNonMatch` 的语义是"规则没匹配上，就把域名解析成 IP 再匹配一次"，副作用是**出口也拿那个 IP 去连**
改成 `AsIs` 之后域名原样交给上游、由上游自己解析，本地 DNS 再怎么被污染都影响不到已代理的流量

**③ 默认解析换成 DoH（推荐，但要分级）**：

```json
"dns": {
  "servers": ["https://1.1.1.1/dns-query", "https://dns.google/dns-query"],
  "queryStrategy": "UseIPv4"
},
"outbounds": [{ "tag": "dns-out", "protocol": "dns" }, ...],
"routing": { "rules": [
  { "type": "field", "inboundTag": ["dns-in"], "outboundTag": "dns-out" },
  ...
] }
```

**必须分级**：`openai/chatgpt/anthropic` 那几个域名继续走 `223.5.5.5`（上一节要的就是假地址），
其余域名才走 DoH。**全量 DoH 会把上一节的解法弄坏**——ChatGPT 立刻又变「地区不可用」。

### 8.2 顺带发现的另一个坑：分流把图床分错了线

同期还发现 `pbs.twimg.com` / `video.twimg.com` 被规则单独分给了 **hysteria2** 出站，而其它域名走 vless
症状同样是"文字活、图片死"。改法：把它们从 hy2 规则里摘掉，落回默认出站；hy2 留给 YouTube / Netflix

### 8.3 排查顺序（下次直接照这个走）

```sh
1) dig +short A <出问题的域名> @<旁路由>     # 先看解析真假
2) 看 routing.rules 里这个域名被分给哪个出站    # 再看分流
3) 用旁路由自己的 socks 口实测一次：
   curl -x socks5h://127.0.0.1:10808 https://<域名>/  -o /dev/null -w '%{http_code}\n'
   —— 旁路由通、手机不通 → 问题在嗅探/域名还原/终端缓存
4) 手机杀掉 App 重开（会清掉失败的缓存状态）
```

---

## 九、一句话总结

> 不是 IP 被封，也不是代理没走通
> 是 iOS App 在被判定"不可信"时，**给它一个假的解析结果、让域名被嗅探还原**，
> 连接落到另一个边缘上，判定就变了
> 玄学，但可复现
