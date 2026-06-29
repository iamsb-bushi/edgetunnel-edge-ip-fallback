# 修改版说明

本仓库基于原项目：

```text
cmliu/edgetunnel
https://github.com/cmliu/edgetunnel
```

本修改版继续保留原项目的 GPL-2.0 开源协议。发布、复制、分发或再次修改本项目时，请继续保留 `LICENSE`、源码和原项目说明。

## 是否需要 @ 原作者

不强制需要在 GitHub 里 `@` 原作者。

更推荐的做法是：

- 保留原项目 README 中的原作者和项目链接。
- 保留 `LICENSE`。
- 在本文件中说明这是基于 `cmliu/edgetunnel` 的修改版。
- 如果发布到 GitHub，仓库描述或 README 中写明 `Based on cmliu/edgetunnel` 即可。

如果你想礼貌通知原作者，可以在 release、issue 或 README 中提到 `cmliu/edgetunnel`。但直接 `@` 作者会触发通知，不是协议要求。

## 本修改版主要变化

- ECH 链接生成改为使用当前节点域名，并使用 `udp://223.5.5.5:53`。
- ECH 启用时，VLESS / Trojan 订阅节点的 TLS `host`、`sni` 和 ECH 仍使用配置域名。
- 支持通过 `SUB_NODE_ADDRESS`、`SUB_EDGE_IP`、`EDGEIP`、`PREFERRED_IP` 指定单个订阅连接目标。
- 支持通过 `SUB_EDGE_IPS` 或 `SUB_NODE_ADDRESSES` 指定多个 Cloudflare 边缘 IP。
- 当设置多个边缘 IP 时，订阅会为每个 IP 生成一条节点，而不是只选择其中一个。
- 多边缘 IP 订阅会额外生成一条 `domain` 域名兜底节点。
- 非 SS WebSocket 链接固定使用 `alpn=http/1.1`，避免传统 WebSocket 误协商到不兼容的 HTTP/2。
- 默认单节点链接也应用相同的 ECH 和 ALPN 逻辑。
- 增加部分 `request.cf` 字段保护，避免在非标准请求环境中报错。

## 边缘 IP 节点

关于 `SUB_EDGE_IPS` / `SUB_NODE_ADDRESSES` 的配置方式、Cloudflare 边缘 IP 的获取方式，以及 `domain` 兜底节点的作用，请看：

[边缘 IP 与域名兜底说明](./EDGE_IPS.md)

## 隐私与发布注意

这个发布包不应包含：

- Cloudflare 账号 ID
- KV namespace ID
- 私有域名
- 管理密码
- UUID
- 订阅 token
- 本地备份文件
- `.wrangler`
- `.git`

发布前需要自行配置：

- `wrangler.toml` 中的 Worker / Pages 项目信息
- KV namespace 绑定 `KV`
- `ADMIN` 或 `PASSWORD` 等管理变量
- 可选的自定义域名 `HOST`
- 可选的边缘 IP 变量 `SUB_EDGE_IPS`
