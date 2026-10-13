# 排查手册

Shadowrocket 配置在使用中遇到的运行时问题与排查方法。配置约定见 [CLAUDE.md](../CLAUDE.md)，v2rayN 端的排查记录见 [v2rayn.md](v2rayn.md)，本文件只记录运行时排查。

排查原则：**代理异常先查本文件，避免误改分流规则**。很多"看起来像分流问题"的症状实为系统层原因，动规则文件无效。

## BlockHttpDNS 拦截导致依赖 HTTPDNS 的国内 App 卡顿

**症状**：美团外卖等 App 页面加载明显变慢（首屏卡顿）；关闭代理或换配置后恢复。浏览器等不依赖内置 HTTPDNS 的应用不受影响。

**根因**：`shadowrocket.conf` 曾加入 `BlockHttpDNS` 拦截（`RULE-SET, ...BlockHttpDNS.list, REJECT`），其中包含 `DOMAIN,httpdns.meituan.com`。美团 App 内置 HTTPDNS 请求被 REJECT 后，SDK 要等握手失败/超时才回落系统 DNS，首屏因此明显变慢。**同类问题会出现在任何重度依赖内置 HTTPDNS 的国内 App**（支付宝、腾讯系等）。

**处置**：已移除 `BlockHttpDNS` 拦截（提交 `fd6cc5d`）。国内 App 回落系统 DNS 后走 `dns-server` 的阿里/腾讯 UDP，解析与 CDN 调度无损，防泄露目标不受影响。

**相关排查**：同批加入的 `block-quic = all-proxy` 曾先被怀疑为元凶，实测排除（移除后仍慢——美团的 QUIC 走直连，`all-proxy` 不拦直连流量），已恢复（提交 `5128f97`）。

### 排查路径回顾（避免重走弯路）

| 线索 | 指向 |
|---|---|
| 移除 block-quic 仍慢 | QUIC 屏蔽不背锅——all-proxy 只拦走代理的 QUIC |
| 移除 BlockHttpDNS 后恢复 | HTTPDNS 被 REJECT → App 回落系统 DNS 前的超时等待是元凶 |
| 美团主域被 ACL4SSR 覆盖走直连 | 流量未漏走代理，无需改分流规则 |

**教训**：App（尤其国内大厂 App）卡顿时，先查 `BlockHttpDNS` 拦截——它置于 `[Rule]` 最前、拦截面最大，且症状（卡顿而非断流）容易被误判为网络或分流问题。

## `dns-server` 中境外 DoH 未实际生效（DNS 泄漏实测）

**背景**：issue #2 质疑 `dns-server` 混排「国内 UDP + 境外 DoH」的写法：若 Shadowrocket 并发竞速，明文 UDP 几乎恒快于 DoH，DoH 起不到防污染作用，反而让 Cloudflare/Google 看到国内直连域名的查询。该行为无官方文档背书（Surge 语义是「配置加密 DNS 后明文服务器不再用于普通域名」，与 Shadowrocket 实测不符），需实测裁决。

**测试方法**：用 DNS 泄漏测试站（dnsleaktest.com Extended test）观察权威 NS 侧的 resolver 名单。原理：测试站解析**唯一随机子域名**，缓存必然 miss，本地每一路参与查询的 DNS 服务器都必然去问权威 NS、留下出口 IP 记录——**权威侧名单 = 实际参与查询的完整名单**，比翻本地日志可靠。关键前置：给测试站临时加 `DOMAIN-SUFFIX,dnsleaktest.com,DIRECT` 规则，让解析发生在本地 `dns-server`，而非落 `FINAL` 被代理远端解析。

**结果**（两轮，上海电信家宽）：

| 轮次 | `dns-server` 配置 | 权威侧 resolver |
|---|---|---|
| 1 | 阿里 + 腾讯 UDP + Cloudflare/Google DoH | 阿里出口、上海电信 IP 若干，**无任何 Cloudflare/Google 出口** |
| 2 | 仅阿里 UDP | 阿里出口 + `180.153.91.x` 等上海电信 IP |

**结论**：

1. **两个境外 DoH 从未参与任何查询**——若参与过，随机子域名必然在权威侧留下它们的出口 IP。1.1.1.1/8.8.8.8 的 443 从国内直连不稳，DoH 建连大概率从未成功。属死配置，已移除（`shadowrocket.conf` 的 `dns-server` 仅保留阿里/腾讯，`[Host]` 中配套的 `dns.google` bootstrap 映射一并移除）。
2. **防污染实际由规则层承担**，与 DNS 服务器选择无关：IP 规则全带 `no-resolve`，境外域名不经本地解析，落 `FINAL` 后由 `force-remote-dns` 交代理远端解析。
3. **本地明文 UDP 解析对运营商可见**是固有属性（测试中权威侧出现上海电信出口 IP）。其中 `180.153.91.x` 倾向为阿里/DNSPod 在电信 IDC 的托管节点（与公开部署信息吻合，DNSPod 出口在去掉 `119.29.29.29` 后即消失），也不排除运营商介入明文查询，未做进一步裁决——无论哪种情况均不影响上述结论与处置。
4. 顺带修正注释措辞：CDN 调度准来自公共 DNS **按递归源 IP 调度**，普通客户端的 UDP 查询并不携带 ECS。

**遗留风险**：**代理节点域名是唯一必须本地解析的境外域名**（明文 UDP 有被污染的理论风险，症状为连不上节点）。缓解：节点用纯 IP，或在 `[Host]` 中静态映射节点域名（命中映射即跳过 DNS 查询，连查询都不发）。

**教训**：判断 DNS 配置是否生效，**别看配置宣称什么，看权威侧谁查过**——随机子域名泄漏测试一次给出完整的 resolver 名单，本地日志只能看到「谁赢了」，看不到「谁根本没上场」。

## 影视聚合站视频无法播放（第三方资源域名未收录）

**症状**：影迷窝（`yingmiwo.me`）等影视聚合站，开代理时**网页能正常打开，但视频始终加载不出来**；关闭代理即可正常播放。网页与视频流表现不一致。

**根因**：这类站点的**网页域名与视频资源域名不是同一个**。站点只是壳，真正的视频走第三方采集源——本次实测为「极速资源」（`jisu` 系列）：

| 域名 | 用途 |
|---|---|
| `yingmiwo.me` | 站点页面、播放信息接口 |
| `vv.jisuzyv.com` | 播放列表（m3u8） |
| `p.jisuts.com` | 视频分片（`.ts`） |
| `img.jisuimage.com` | 封面图 |

这些资源域的 **NS 在阿里云（`vip7/vip8.alidns.com`），但 A 记录指向境外 IP**（如 `185.34.146.254`）。它们未被任何规则集收录，于是：

1. `GEOIP,CN,DIRECT,no-resolve` **带 `no-resolve`，不会为域名触发 DNS 解析**，直接跳过
2. 请求落到最末的 `MATCH` → 走代理（美国节点）
3. **服务端拒绝来自代理的连接，TLS 握手直接失败**（实测 `SEC_E_ILLEGAL_MESSAGE`）
4. 视频加载不出来

关代理时家宽 IP 直连，服务端放行（实测 HTTP 200），故能正常播放。

**处置**：三个资源域已收录至 `rules/ChinaDirect.list` 末尾特例区。

### 关键机制：`no-resolve` 决定了「域名是否被收录」才是分水岭

这是本次问题的核心。配置文件末尾两条兜底：

```
GEOIP,CN,DIRECT,no-resolve     ← 不解析域名
MATCH,<代理组>                  ← 一切未匹配的落这里
```

`no-resolve` 的含义是**不为了匹配 IP 规则而触发 DNS 解析**。因此：

> 任何**域名**请求，只要不在 `ChinaDirect` / `ChinaDomain` / `ChinaMedia` 等**域名类规则集**里，**无论解析到国内还是境外 IP，都会直接落到 `MATCH` 走代理**。

也就是说，一个域名走不走代理，取决于**它有没有被域名规则集收录**，而不取决于它的 IP 归属。这与「国内 IP 直连」的直觉相反，是本类问题的认识起点。

### 定位方法（可复用）

**不要靠推测，直接抓实际数据流**——影视聚合站的域名至少有「站点 / 列表 / 分片」三组，猜不出来。

**第一步：找出站点的播放信息接口**

用无头 Chrome 记录页面请求：

```bash
chrome --headless=new --log-net-log=netlog.json \
       --user-data-dir=<临时目录> --virtual-time-budget=30000 \
       --dump-dom "<播放页 URL>" > /dev/null
grep -oE '"url":"https?://[^"/]+' netlog.json | sed 's/"url":"//' | sort -u
```

本次抓到：`https://yingmiwo.me/api/filmPlayInfo?id=...&playFrom=...&episode=...`

**第二步：调用该接口看返回**

该接口需登录态，**在已登录的浏览器地址栏直接打开即可**。返回内容会直接给出视频地址：

```json
{"current": {"episode": "第102集",
             "link": "https://vv.jisuzyv.com/play/elYpPV1a/index.m3u8"}}
```

**第三步：拉取 m3u8 找出分片域名**

```bash
curl -s --noproxy '*' "https://vv.jisuzyv.com/play/elYpPV1a/index.m3u8" | grep -oE 'https?://[^" ]+'
```

分片（`.ts`）**往往在另一个域名上**（本次为 `p.jisuts.com:999`），必须一并收录，否则视频仍会卡在加载分片这一步。

**第四步：对比直连与代理的差异**

```bash
curl -s -o /dev/null --noproxy '*'            -w "直连 HTTP:%{http_code}\n" "https://vv.jisuzyv.com/..."
curl -s -o /dev/null -x http://127.0.0.1:7897 -w "代理 HTTP:%{http_code}\n" "https://vv.jisuzyv.com/..."
```

### ⚠️ 测试陷阱：`https_proxy` 环境变量会让 curl 静默走代理

本机 Git Bash 环境预设了 `https_proxy=http://127.0.0.1:7897`，curl 会**自动读取并走代理**。不加 `--noproxy '*'` 的所谓「直连测试」实际上走了代理，结论会被完全带偏。

测试前先用 `curl -v` 确认实际路径，输出中会有：

```
* Uses proxy env variable https_proxy == 'http://127.0.0.1:7897'
```

**任何直连/代理对比测试，都必须先确认这一点。**

### 排查路径回顾（避免重走弯路）

本次先后做过两次错误归因，共同点是**用间接线索推测，而非抓实际数据**：

| 错误归因 | 当时的依据 | 为什么错 |
|---|---|---|
| 「站点排斥代理出口 IP」 | 站点开 Cloudflare 挑战、whois 隐私保护、经代理耗时异常 | 方向沾边但没定位到具体域名；据此加 `yingmiwo.me` 直连，对视频毫无作用 |
| 「夸克网盘未收录」 | 站点 JS 里出现 `pan.quark.cn`、存在 `panparse` 子域 | JS 引用只说明站点**支持**该网盘；本例影片实际走「极速资源」，与夸克无关 |

**教训**：影视聚合站的「站点域名 / 播放列表域名 / 分片域名」通常是三组不同的域，**只看站点或 JS 里的片段线索必然误判**。应尽早抓播放接口的返回内容——那里有权威答案，一步到位。

第二个教训在测试方法上：`https_proxy` 环境变量会让 curl 静默走代理，**直连测试不显式 `--noproxy '*'` 等于没测**。

## hash 路由页面经代理打开报 400「The request URL is invalid.」（fragment 未剥离）

**症状**：微信内置浏览器打开漫步者售后服务页（易售后平台，`http://www.eshouhou.cn/html5/#/pages/login/startPage/startPage?…`）时，开代理显示一张 `HTTP Error 400. The request URL is invalid.` 错误页，关闭代理立即正常。Safari 打开同一链接不受影响。Shadowrocket 日志里只有 `DIRECT` 行，**既无代理记录也无拦截记录**，分流规则看不出任何异常。

**根因**：该页是 hash 路由，`#` 之后是 fragment。按 RFC 3986 §3.5，客户端必须剥离 fragment 后才发请求；开代理时这条规范化在转发链上丢失，含 `#` 的完整 URL 被放进了 HTTP 请求行。源站为 `Microsoft-IIS/10.0` + `Microsoft-HTTPAPI/2.0`（http.sys），http.sys 严格校验请求行 URL 形态，遇到 `#` 判定非法并返回该错误页。

### 关键判据：用 http.sys 的 400 文案反推病因

该站的 400 文案按触发条件严格区分，文案本身就是最快的定位线索：

| 请求行 | 文案 |
|---|---|
| 缺 `Host` 头 | `HTTP Error 400. The request hostname is invalid.` |
| 含空格 | `HTTP Error 400. The request is badly formed.` |
| 含 `#`，或含裸 `%` / `%zz` 等非法转义 | `HTTP Error 400. The request URL is invalid.` |

对源站直接发请求行的实测对照（`101.201.148.74:80`）：

```
GET /html5/#/pages/login/…              → 400 The request URL is invalid.
GET http://www.eshouhou.cn/html5/#/…    → 400 The request URL is invalid.
GET /html5/#                            → 400 The request URL is invalid.
GET /html5/                             → 200 OK   ← 按规范剥离 fragment
GET /html5/?access_token=…（无 #）       → 200 OK   ← 带完整 query（含 Base64 尾部 =），无 fragment
```

### 判据：这三个现象都与分流无关

下列现象容易被读成分流问题，实际都属于本机制的必然结果：

| 现象 | 实际含义 |
|---|---|
| 日志只有 `DIRECT`，无代理/拦截行 | 连接确实直连且已建立；出错在应用层报文的 URL 形态，路由决策全程正常 |
| 命中的规则与其他客户端一致 | 域名由上游 `ChinaDomain.list` 的 `DOMAIN-SUFFIX,cn` 给出直连，规则层未参与 |
| 关代理即恢复 | 关代理后客户端按规范剥离 fragment，请求行变为 `/html5/` |

> 开启代理改变的不只是路由，还可能是**请求报文本身**。只查规则文件与 IP 归属覆盖不到后一类。

**处置**：

- 改用 **HTTPS** 打开该页：HTTPS 下代理只看到 `CONNECT host:443`，请求路径在 TLS 内由浏览器构造，fragment 必然被剥离，问题从根上绕开。
- 该站对 `/html5/` **不做 HTTP→HTTPS 跳转**（`http` 与 `https` 均返回 200），拿不到服务端自动保护，需手动把链接的 `http://` 改为 `https://`。
- 临时关闭代理可访问，仅作应急。
