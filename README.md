# everret-knowledge-base

这是everret的知识库，整理关于ctf的知识与payload，用于事后复盘

## 目录结构

```
everret-knowledge-base/
├── README.md                  # 本文件：总入口
├── .gitignore                 # 忽略规则（大附件、密钥、草稿）
├── .gitattributes             # 换行符规则（跨设备一致）
├── docs/
│   └── Git使用手册.md          # Git 配置与日常命令手册（含本机踩过的坑）
├── web/                       # Web 方向
│   ├── README.md              # 方向总览 + 考点进度表
│   ├── sqli/                  # ┐
│   │   ├── 原理笔记.md         # │ 每个考点固定三件套
│   │   ├── payload.md         # │
│   │   └── 做过的题.md         # ┘
│   ├── xss/
│   ├── ssti/
│   ├── ssrf/
│   ├── upload/                # 文件上传
│   └── rce/                   # 命令执行
├── crypto/                    # 密码学（README 里有考点清单与工具表）
├── misc/                      # 杂项：流量分析、隐写、取证…
└── review/                    # 比赛复盘，一场一个文件
    ├── README.md              # 复盘索引
    ├── 复盘模板.md             # 复制它来写新复盘
    └── 2025-XX杯.md           # 实际复盘
```

## 快速导航

| 我想…… | 去哪 |
| --- | --- |
| 查某个漏洞的**原理** | `web/<考点>/原理笔记.md` |
| 比赛时**直接抄 payload** | `web/<考点>/payload.md` |
| 看**以前做过什么题** | `web/<考点>/做过的题.md` |
| 写**比赛复盘** | 复制 `review/复盘模板.md` |
| 学**Git 怎么用**（配置/日常/排错） | [docs/Git使用手册.md](docs/Git使用手册.md) |
| 加一个**新考点** | 复制现成考点目录的三件套，再在 `web/README.md` 登记一行 |

## 写作约定

1. **每个考点三个文件**，职责不混：原理写清"为什么"，payload 写清"怎么打"，做过的题写清"当时怎么想通的"。
2. **payload 必须写适用条件**（数据库/版本/权限/前提），否则日后不敢用。
3. **做过的题记录格式统一**：日期 / 题目 / 考点 / 突破口 / 关键 payload / 卡在哪 / 结论。**"卡在哪"和"结论"最有价值。**
4. **大附件不入库**：pcap、内存镜像、压缩包等只放链接，仓库里留命令和结论。
5. **提交信息用前缀**：`feat` / `fix` / `docs` / `review` / `chore` / `refactor`，具体见 Git 手册第 7.2 节。
6. **一句话木马只用 base64 记录**：`@ev` + `al($_POST[1])` 这种一字不差的写法会被 Windows Defender 判定为病毒并**锁死文件**，`git add` 会报 `Invalid argument`。本仓库统一用 base64（旁边给解码命令），详见 [Git 手册 8.10 节](docs/Git使用手册.md#810-error-open-invalid-argument文件被杀毒软件锁死)。

## 日常三连

```bash
git pull --rebase                        # 开工：先同步
git add -A && git commit -m "feat(web/sqli): 补充报错注入函数"
git push                                 # 收工：推上去备份
```

## 免责说明

本仓库所有内容仅用于**合法授权的安全测试**、**CTF 竞赛**与**个人学习研究**。
对未获得授权的系统使用其中的技术属于违法行为，作者不承担任何由此产生的责任。
