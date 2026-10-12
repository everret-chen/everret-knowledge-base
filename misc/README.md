# Misc 杂项

> 流量分析、隐写、取证、编码、沙箱逃逸、AI/区块链题目等等都归这里。按**题型**建子目录。

## 目录约定

```
misc/
├── README.md              # 本文件
├── traffic/               # 流量分析（pcap）
│   ├── 原理笔记.md
│   ├── payload.md         # 常用命令（tshark/过滤器）
│   └── 做过的题.md
├── stego/                 # 隐写（图片/音频/文件）
├── forensics/             # 取证（内存/磁盘/日志）
├── encoding/              # 编码与转换套路
└── ...
```

> ⚠️ 大附件（pcap、内存镜像、压缩包）默认被 `.gitignore` 忽略。**只提交命令与结论**，附件放网盘并在 `做过的题.md` 里贴链接。确实要入库用 `git add -f` 或 Git LFS，详见 [docs/Git使用手册.md](../docs/Git使用手册.md)。

## 考点清单

| 题型 | 子目录 | 常用工具 |
| --- | --- | --- |
| 流量分析 | `traffic/` | Wireshark、tshark、NetworkMiner、`binwalk` |
| 图片隐写 | `stego/` | `zsteg`、`steghide`、`stegsolve`、`exiftool`、`pngcheck` |
| 音频隐写 | `stego/` | Audacity、`sox`、频谱图 |
| 文件隐写/拼接 | `stego/` | `binwalk`、`foremost`、`dd`、`file`、`xxd` |
| 内存取证 | `forensics/` | Volatility 2/3、`strings` |
| 磁盘/镜像取证 | `forensics/` | `mmls`、`fls`、`icat`、Autopsy |
| 日志分析 | `forensics/` | `grep`/`awk`、`jq`、Excel |
| 编码/进制 | `encoding/` | CyberChef、Python |
| 压缩包 | `archive/` | `zip2john`、`hashcat`、`bkcrack`（明文攻击） |
| 沙箱逃逸 / 逃逸类 | `sandbox/` | 视题目而定 |

## 通用套路（不知道从哪下手时按顺序试）

```bash
file 附件                 # ① 永远先看真实类型，别信后缀
xxd 附件 | head           # ② 看文件头有没有藏东西
binwalk 附件              # ③ 找拼接/嵌套的文件
strings -n 8 附件 | less  # ④ 找可读字符串与 flag 形状
exiftool 附件             # ⑤ 看元数据（作者、注释、GPS）
foremost -i 附件 -o out   # ⑥ 分离文件
```

## 记录习惯

misc 题最容易"做出来就忘"。`做过的题.md` 里**务必写清用哪条命令发现的**，因为下次遇到同类题靠的是那条命令，不是结论。
