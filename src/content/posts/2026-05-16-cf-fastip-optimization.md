---
title: Cloudflare IP优选完全指南：让国内访问速度提升
published: 2026-05-16
description: 详细介绍Cloudflare IP优选的原理、多种实现方案以及针对不同服务的配置方法，显著提升网站在国内的访问速度和可用性。
image: /pic/cf-fastip/cf-fastip-11.avif
tags: [Cloudflare, CDN, 网络优化, 教程]
category: 技术教程
draft: false
---

本文搬运自 https://2x.nz/posts/cf-fastip/，原文作者：二叉树树，原文为公开可转载内容，已做二次优化与结构调整。

## 前言

Cloudflare在国内的访问速度一直是许多开发者的痛点。官方分配的IP在国内访问时延迟往往较高，甚至可能出现无法访问的情况。通过IP优选技术，我们可以手动将域名解析到国内访问更快的Cloudflare节点，从而显著提升网站的访问速度和可用性。

从对比测试可以看到，优选过的网站响应速度有很大提升，出口IP也变多了。这不仅让网站可用性大大提高，加载速度也显著变快。

#### 未优选

![未优选效果](/pic/cf-fastip/098f9ee71ae62603022e542878673e19bdcaf196.avif)

#### 已优选

![已优选效果](/pic/cf-fastip/cf-fastip-11.avif)

## 什么是优选

优选的核心就是选择一个国内访问速度更快的Cloudflare节点IP或域名。要实现优选，我们需要做到两点：

1. 自己控制路由规则
2. 自己控制DNS解析

通过Cloudflare SaaS或Worker路由，我们可以同时实现这两点。

## 优选原理

CDN通过不同域名给不同内容提供服务，可以抽象为两层：规则层和解析层。

当我们在Cloudflare添加一条开启了小黄云的解析时，Cloudflare会为我们做两件事：
- 帮我们写一条DNS解析指向Cloudflare
- 在Cloudflare创建一条路由规则

优选的本质是手动更改DNS解析，使其指向一个更快的Cloudflare节点。但一旦关闭小黄云，路由规则也会被删除，这会导致访问异常。

而SaaS和Worker路由的出现改变了这一切。使用SaaS后，Cloudflare不再自动做这两件事：
- 你可以自己写一条SaaS规则（规则层）
- 你可以自己写一条CNAME解析到优选节点（解析层）

使用Worker路由同理，创建Worker路由规则后，DNS解析就可以随意指向任何优选节点了。

## 选择优选域名

### 传统优选域名

常用的社区优选域名：https://cf.090227.xyz

这些优选域名通过扫描Cloudflare官方IP段，找出国内延迟最低的IP整理而成。

### Cloudflare Byoip 优选

#### 什么是Byoip？

Cloudflare Byoip（Bring Your Own IP），即如果用户自己拥有一个IP或IP段，可以将其托管给Cloudflare，并使其受益于Cloudflare全球网络的加速与安全。

简单说，有一些IP不直接隶属于Cloudflare，但我们CNAME到这个IP后仍然可以正常访问到部署在Cloudflare上的服务。这些IP可能并不是Anycast，但国内延迟可能会明显优于Cloudflare的官方IP段。

#### 如何找到Cloudflare Byoip？

可以前往 https://ipregistry.co/AS209242#ranges 查看AS209242（Cloudflare London, LLC）的IP段。

使用ITDog等工具强制绑定IP访问你的Cloudflare服务，不返回403即可使用。

![ITDog测试](/pic/cf-fastip/838f685e-3913-4b21-995e-5ee149f4bffa.avif)

#### 注意事项

- 一些Byoip可能会强制跳转到自己的网站，需要查看ITDog的测试日志是否有重定向，别让你的网站成为他人的引流站
- 建议设置一个机器定时筛选不可用的IP，并添加一些Cloudflare官方IP段作为备选，防止服务宕机

## 各类优选方案

### 1. Worker项目优选（最简单）

如果你需要优选Page/Worker项目：

1. 如果是Page项目，先将项目转为Worker
2. 编写Worker路由，填写`你的域名/*`

![Worker路由配置](/pic/cf-fastip/cf-fastip.avif)

3. 写一条DNS解析到想要的优选域名

![DNS解析](/pic/cf-fastip/cf-fastip-1.avif)

**优势**：不需要折腾SaaS，不需要多域名。

### 2. Worker路由反代全球并优选（进阶）

原理：通过Worker反代源站，然后将Worker的入口节点进行优选。源站接收到的Host头仍然直接指向源站的解析。

创建一个Cloudflare Worker，写入以下代码：

```javascript
// 域名前缀映射配置
const domain_mappings = {
  '源站.com': '最终访问头.',
  // 例如：
  // 'gitea.072103.xyz': 'gitea.',
  // 则设置Worker路由为gitea.*都会反代到gitea.072103.xyz
};

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  const current_host = url.host;

  // 强制使用 HTTPS
  if (url.protocol === 'http:') {
    url.protocol = 'https:';
    return Response.redirect(url.href, 301);
  }

  const host_prefix = getProxyPrefix(current_host);
  if (!host_prefix) {
    return new Response('Proxy prefix not matched', { status: 404 });
  }

  // 查找对应目标域名
  let target_host = null;
  for (const [origin_domain, prefix] of Object.entries(domain_mappings)) {
    if (host_prefix === prefix) {
      target_host = origin_domain;
      break;
    }
  }

  if (!target_host) {
    return new Response('No matching target host for prefix', { status: 404 });
  }

  // 构造目标 URL
  const new_url = new URL(request.url);
  new_url.protocol = 'https:';
  new_url.host = target_host;

  // 创建新请求
  const new_headers = new Headers(request.headers);
  new_headers.set('Host', target_host);
  new_headers.set('Referer', new_url.href);

  try {
    const response = await fetch(new_url.href, {
      method: request.method,
      headers: new_headers,
      body: request.method !== 'GET' && request.method !== 'HEAD' ? request.body : undefined,
      redirect: 'manual'
    });

    // 复制响应头并添加CORS
    const response_headers = new Headers(response.headers);
    response_headers.set('access-control-allow-origin', '*');
    response_headers.set('access-control-allow-credentials', 'true');
    response_headers.set('cache-control', 'public, max-age=600');
    response_headers.delete('content-security-policy');
    response_headers.delete('content-security-policy-report-only');

    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: response_headers
    });
  } catch (err) {
    return new Response(`Proxy Error: ${err.message}`, { status: 502 });
  }
}

function getProxyPrefix(hostname) {
  for (const prefix of Object.values(domain_mappings)) {
    if (hostname.startsWith(prefix)) {
      return prefix;
    }
  }
  return null;
}
```

创建Worker路由后，写一条DNS解析：

![创建路由](/pic/cf-fastip/56752d54-26a5-46f1-a7d9-a782ad9874cb.avif)

![路由配置](/pic/cf-fastip/d025398c-39e3-4bd7-8d8f-2ce06a45007d.avif)

```
CNAME gitea.afo.im --> 优选域名
```

### 3. Cloudflare R2 优选

1. 创建R2实例

![创建R2实例](/pic/cf-fastip/cf-fastip-4.avif)

2. 绑定一个自定义域

![绑定自定义域](/pic/cf-fastip/cf-fastip-5.avif)

3. 前往域名 → 规则 → Cloud Connector

![Cloud Connector](/pic/cf-fastip/cf-fastip-6.avif)

![配置Cloud Connector](/pic/cf-fastip/cf-fastip-7.avif)

![Cloud Connector配置](/pic/cf-fastip/cf-fastip-8.avif)

4. 写一条解析指向优选域名：
   ```
   fast-r2.2x.nz CNAME cf.090227.xyz
   ```

### 4. 传统SaaS优选

#### SaaS做了什么？

Cloudflare SaaS是一个不需要改变域名NS服务器，就可以让其受益于Cloudflare网络的功能。

当一个域名被SaaS到一个已经在Cloudflare的域名后，它就完整受益所有Cloudflare服务。如我将 umami.acofork.com SaaS 到 2x.nz，我就可以在 2x.nz 里为 umami.acofork.com 写规则了：

![SaaS配置示例1](/pic/cf-fastip/cf-saas-1.avif)

![SaaS配置示例2](/pic/cf-fastip/cf-saas-2.avif)

![SaaS配置示例3](/pic/cf-fastip/cf-saas-3.avif)

Worker中的路由规则也适用：

![Worker路由规则](/pic/cf-fastip/cf-saas-4.avif)

#### SaaS优选步骤

**准备工作**：需要一个域名或两个域名（单域名直接用子域名即可）。

**具体步骤**：

1. 新建DNS解析，指向你的源站，开启CF代理

![DNS解析配置](/pic/cf-fastip/c94c34ee262fb51fb5697226ae0df2d804bf76fe.avif)

2. 前往辅助域名的SSL/TLS → 自定义主机名，设置回退源为第一步的DNS解析域名（推荐HTTP验证）

3. 添加自定义主机名，选择自定义源服务器，填写第一步的域名

![自定义主机名配置](/pic/cf-fastip/f6170f009c43f7c6bee4c2d29e2db7498fa1d0dc.avif)

4. 在辅助域名添加一条解析，CNAME到优选节点，不开启CF代理

![优选节点解析](/pic/cf-fastip/4f9f727b0490e0b33d360a2363c1026003060b29.avif)

5. 在主力域名添加解析，域名为自定义主机名，目标为刚才的cdn域名，不开启CF代理

![主力域名解析](/pic/cf-fastip/6f51cb2a42140a9bf364f88a5715291be616a254.avif)

6. 优选完毕，确保优选有效后尝试访问

![优选完成](/pic/cf-fastip/cf-fastip-10.avif)

7. （可选）将cdn子域的NS服务器更改为阿里云、华为云、腾讯云云解析做线路分流解析

**工作流**：
用户访问 → CNAME解析访问cdn域名 → 到达优选域名进行优选 → CF边缘节点识别到携带的源主机名 → 查询发现回退源 → 回退到回退源内容 → 访问成功

**注意**：Cloudflare最近将新接入的域名SSL默认设为了完全，记得将SSL改为灵活。

## 针对不同服务的优选

### Cloudflare Pages

两种方案：

1. 将绑定到Page的子域名的NS服务器更改为其他DNS服务商，做线路分流解析
2. 将Page项目升级为Worker项目，使用Worker优选方案（推荐，更简单）

### Cloudflare Workers

1. 在Workers中添加路由
2. 将路由域名从指向xxx.worker.dev改为优选域名
3. 如果是外域，SaaS后再添加路由：

![Worker路由](/pic/cf-fastip/cf-fastip-12.avif)

![外域路由](/pic/cf-fastip/cf-fastip-13.avif)

### Cloudflare Tunnel（ZeroTrust）

先参照传统SaaS优选设置完毕，源站即为Cloudflare Tunnel。

![Tunnel配置](/pic/cf-fastip/cf-fastip-2.avif)

![Tunnel规则](/pic/cf-fastip/cf-fastip-3.avif)

接下来需要让最终访问的域名打到Cloudflare Tunnel的流量正确路由。创建一个Tunnel规则，域名为最终访问的域名，源站指定和刚才的一致即可。

最后写一条DNS记录：
```
umami.2x.nz CNAME 你的优选域名
```

### 使用了各种CF规则的网站

规则针对于你的最终访问域名即可，因为CF的规则是看主机名的，而不是看由谁提供的。

### 虚拟主机

保险起见，建议将源站和优选域名同时绑定到你的虚拟主机，保证能通再一个个删除。

## 总结

Cloudflare IP优选是提升国内访问速度的有效手段。选择哪种方案取决于你的具体需求：

- **简单场景**：Worker项目优选，最简单直接
- **进阶场景**：Worker路由反代，适合需要统一入口的场景
- **R2存储**：使用Cloud Connector配合优选
- **复杂架构**：传统SaaS优选，适合需要精细控制的场景

所有优选一个域名即可，无需两个域名。设置时注意SSL模式调整，建议定期检查优选IP的可用性。

---

相关视频教程：
- [全解](https://www.bilibili.com/video/BV1QpSoBqERj)
- [SaaS原理](https://www.bilibili.com/video/BV1A5rpBqENh/)
- [Worker/Pages优选](https://www.bilibili.com/video/BV1KNmtB6EU7/)
- [R2优选](https://www.bilibili.com/video/BV115KLzSEiv/)
- [Tunnel优选](https://www.bilibili.com/video/BV1WGMAznEBd)
- [自建优选](https://www.bilibili.com/video/BV1H38vzoEcq/)