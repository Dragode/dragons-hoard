+++
date = '2026-02-25T13:22:15+08:00'
draft = false
title = 'Claude Code Remote Control 不是内网穿透'
tags = ['AI', 'Claude Code', '安全']
categories = ['技术调研']
+++

今天研究了 Claude Code 的 Remote Control 功能，主要想搞清楚一个问题：它到底是不是内网穿透？公司安全条例对穿透类工具管得很严，得确认清楚才敢用。

## 结论先行：不是内网穿透，是出站轮询

官方文档写得很明确——"outbound HTTPS requests only, never opens inbound ports"。本机 Claude Code 进程主动向 Anthropic API 发 HTTPS 请求轮询任务，手机/浏览器端也是连 Anthropic 服务器，Anthropic 在中间做消息路由。整个过程没有开放任何入站端口，没有建立反向隧道，外部无法直连你的机器。

## 架构示意

```
本机 (公司内网)          Anthropic API           手机/浏览器
Claude Code 进程  ──→   云端中继服务器    ←──   claude.ai/code
  出站 HTTPS 轮询         消息路由转发          正常 HTTPS 请求
  ↑ 无入站端口             全程 TLS 加密
  ↑ 无隧道/穿透
```

## 跟真正的内网穿透对比

跟真正的内网穿透（ngrok、frp、Cloudflare Tunnel）对比一下就很清楚：

| 维度 | 内网穿透 | Claude Remote Control |
|------|---------|----------------------|
| 入站端口 | 通过隧道暴露本地端口到公网 | 不开放任何入站端口 |
| 连接方向 | 建立反向隧道，外部可直接访问内网服务 | 仅出站轮询，外部无法直连本机 |
| 网络可达性 | 让内网服务对公网可达 | 本机不对外暴露 |
| 类比 | 在防火墙上开了个洞 | 和用飞书/钉钉收消息一样 |

## 凭证设计

凭证设计也值得一提：用的是多重短期凭证（short-lived, narrowly scoped credentials），每个凭证只管一件事且独立过期，单个泄露不会连带影响。

## 真正该关注的风险

对话内容、代码片段、命令执行结果都会经 Anthropic 境外服务器中继——不过这个风险在普通 Claude Code 使用时就已经存在了，Remote Control 没有增加增量风险。

一个有意思的细节：Remote Control 目前只对 Pro/Max 个人计划开放，不支持 Team/Enterprise，说明 Anthropic 自己也认为它还没达到企业级安全标准。

## 合规性速查

| 常见条例 | 是否违反 |
|---------|---------|
| 禁止内网穿透 | **不违反** — 无穿透，无入站端口 |
| 禁止开放端口 | **不违反** — 仅出站 HTTPS (443) |
| 禁止使用 VPN/隧道 | **不违反** — 无隧道 |
| 代码/数据外传 | **需评估** — 但这是 Claude Code 整体的问题，非 Remote Control 特有 |
| 敏感信息经境外服务器 | **需评估** — 同上 |

## 小结

网络安全条例里"禁止内网穿透"这条不会被触发，但"数据跨境传输"和"未审批 SaaS"这两条需要单独评估——当然这是 Claude Code 整体的问题，不是 Remote Control 特有的。

> 参考来源：[Remote Control 官方文档](https://code.claude.com/docs/en/remote-control)、[Security 官方文档](https://code.claude.com/docs/en/security)
