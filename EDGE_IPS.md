# 边缘 IP 与域名兜底说明

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
