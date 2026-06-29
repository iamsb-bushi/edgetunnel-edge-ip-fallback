# 边缘 IP 与域名兜底说明

## 这个修改版修了什么

原版订阅通常直接使用域名作为节点连接地址，客户端最终连接哪个 Cloudflare 边缘 IP 由 DNS 决定。在部分 Windows 网络或客户端环境里，DNS 选中的边缘入口可能间歇性不可用，表现为节点延迟全部 `-1`，而更换订阅或重新解析后又恢复。

这个修改版对订阅生成逻辑进行了以下调整：

- 允许通过环境变量指定一个或多个 Cloudflare 边缘 IP，固定节点的实际连接入口。
- 每个指定 IP 都会生成对应节点，不会只从多个 IP 中选择一个。
- 指定 IP 后仍自动生成一组 `domain` 域名兜底节点。
- 固定 IP 只替换连接地址，TLS `host`、`sni` 和 ECH 身份继续使用 `HOST` 域名，避免证书和域名校验失配。
- 自动去除重复目标，并忽略格式不正确的地址。

实现上，`读取订阅连接目标列表` 读取环境变量，`解析订阅连接目标列表` 校验并去重地址；生成订阅时遍历全部固定目标，并在末尾追加域名目标。因此使用者不需要再修改 `_worker.js`，只需配置环境变量。

这个版本支持在订阅里同时生成两类节点：

- 固定边缘 IP 节点：连接目标是指定的 Cloudflare 边缘 IP。
- 域名兜底节点：连接目标是你的域名，让 Cloudflare / 系统 DNS 自己调度。

这样做的目的，是在某些客户端或网络环境里绕开不稳定的 Cloudflare 边缘，同时保留域名节点作为兜底。

## 基本变量

先设置你的正常域名：

```text
HOST=你的域名
```

例如：

```text
HOST=example.com
```

如果想额外生成固定 Cloudflare 边缘 IP 节点，设置：

```text
SUB_EDGE_IPS=172.67.153.249,104.21.4.98
```

也可以使用等价变量：

```text
SUB_NODE_ADDRESSES=172.67.153.249,104.21.4.98
```

设置多个 IP 后，订阅会为每个 IP 生成一条节点，并额外自动生成一条 `domain` 域名兜底节点。

例如会生成这些连接目标：

```text
172.67.153.249:443
104.21.4.98:443
example.com:443
```

但它们的 TLS 身份仍然都使用你的域名：

```text
host=example.com
sni=example.com
ech=example.com+udp://223.5.5.5:53
```

也就是说，固定 IP 只作为连接入口，证书校验、SNI、Host、ECH 仍然走域名。

## 在 Cloudflare 中指定 IP

打开对应的 Workers 或 Pages 项目，进入：

```text
设置 -> 变量和机密 -> 生产环境
```

添加以下两项。两项都选择 **文本**，不要选择密钥：

| 变量名称 | 示例值 | 类型 | 作用 |
| --- | --- | --- | --- |
| `HOST` | `wdcf.ccwu.cc` | 文本 | TLS、SNI、Host 和 ECH 使用的域名，不要填写 `https://` 或路径 |
| `SUB_EDGE_IPS` | `172.67.153.249,104.21.4.98` | 文本 | 节点实际连接的一个或多个边缘 IP，使用英文逗号或换行分隔 |

只指定一个 IP：

```text
HOST=wdcf.ccwu.cc
SUB_EDGE_IPS=172.67.153.249
```

指定多个 IP：

```text
HOST=wdcf.ccwu.cc
SUB_EDGE_IPS=172.67.153.249,104.21.4.98
```

保存变量后必须重新部署一次，使生产环境的新部署读取这些变量。部署完成后，在 v2rayN、v2rayNG 或其他客户端中更新订阅。

`HOST` 和 `SUB_EDGE_IPS` 不是密码，不需要设为密钥。`ADMIN`、`PASSWORD`、API Token 等敏感值才应使用密钥类型。

## 可以设置多少个 IP

代码没有设置 IP 数量上限。Cloudflare 单个环境变量目前限制为 5 KB，理论上可以容纳数百个 IPv4 地址，但每增加一个 IP 都会成倍增加订阅节点数量，也会增加客户端解析和测速负担。

实际建议设置 2 到 10 个，通常不超过 20 个。固定 IP 不是永久服务器地址，保留少量经过当前网络实测可用的入口比堆入大量地址更稳定。

## IP 怎么获得

### 1. 查询你的域名 A 记录

在 Windows PowerShell 里运行：

```powershell
Resolve-DnsName 你的域名 -Type A
```

例如：

```powershell
Resolve-DnsName example.com -Type A
```

返回的 A 记录可以放进：

```text
SUB_EDGE_IPS=IP1,IP2
```

### 2. 直接测试某个边缘 IP 是否能打开

可以用 `curl --resolve` 保持访问域名不变，但强制连接到某个 IP：

```powershell
curl.exe -I --connect-timeout 5 --max-time 10 --noproxy "*" --resolve example.com:443:172.67.153.249 https://example.com/
```

如果返回正常 HTTP 响应，例如 `200`、`204`、`301`、`302`、`403` 等，说明这个 IP 至少可以作为候选边缘入口。

### 3. 保留域名兜底

如果不设置 `SUB_EDGE_IPS`，订阅只会使用域名：

```text
example.com:443
```

如果设置了 `SUB_EDGE_IPS`，这个版本仍然会自动额外生成一条 `domain` 节点。这样固定 IP 都不好用时，还可以切回域名调度。

## 使用建议

- Cloudflare 边缘 IP 不要当成永久服务器，它只是当前网络环境下的候选入口。
- 如果某个固定 IP 节点不好用，可以在客户端切换到另一个 IP 节点。
- 如果固定 IP 都不好用，可以切换到 `domain` 节点。
- 如果 `domain` 节点更稳定，可以不设置 `SUB_EDGE_IPS`。
- 如果发现新的 IP 更稳定，可以把它加入 `SUB_EDGE_IPS`。

## 示例

假设你的域名是：

```text
example.com
```

查询到两个 Cloudflare A 记录：

```text
172.67.153.249
104.21.4.98
```

那么可以设置：

```text
HOST=example.com
SUB_EDGE_IPS=172.67.153.249,104.21.4.98
```

订阅里会出现类似：

```text
节点名-172.67.153.249
节点名-104.21.4.98
节点名-domain
```

其中 `domain` 就是走 `example.com:443` 的域名兜底节点。
