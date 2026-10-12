# Crypto 密码学

> 本目录按**考点/密码体制**建子目录，每个子目录沿用三件套约定（见 [web/README.md](../web/README.md)）。

## 目录约定

```
crypto/
├── README.md              # 本文件
├── rsa/
│   ├── 原理笔记.md
│   ├── payload.md         # 脚本与攻击脚本（sage/python）
│   └── 做过的题.md
├── aes/
├── hash/
├── classical/             # 古典密码：凯撒、维吉尼亚、栅栏…
└── ...
```

> Python 脚本放 `payload.md` 里用代码块保存，或用 `scripts/` 子目录（记得在 `.gitignore` 里放行）。

## 考点清单

| 考点 | 子目录 | 状态 |
| --- | --- | --- |
| RSA 基础与常见攻击 | `rsa/` | ⬜ 待建 |
| AES / 分组密码模式 | `aes/` | ⬜ |
| 哈希与长度扩展 | `hash/` | ⬜ |
| 古典密码 | `classical/` | ⬜ |
| 数学基础（数论/有限域/椭圆曲线） | `math/` | ⬜ |
| 随机数与伪随机（MT19937、LCG） | `random/` | ⬜ |
| 编码与转换（base 家族、进制、URL） | `encoding/` | ⬜ |

## 常用工具

| 工具 | 用途 |
| --- | --- |
| Python + `pycryptodome` | 密码学运算 |
| SageMath | 数论、格、椭圆曲线（很多 CTF 题离不开） |
| `gmpy2` | 大整数与快速开方 |
| `z3-solver` | 约束求解，反推未知数 |
| CyberChef | 编码转换、快速试探（网页版最好用） |
| `factordb.com` | 查已知的大数分解 |
| RsaCtfTool | RSA 常见攻击自动化 |
| yafu / msieve | 本地整数分解 |

## 起步习惯

1. **先看给了什么**：`n`、`e`、`c`、公钥文件、加密脚本——**有没有加密脚本往往决定思路**。
2. **先算 `e` 是否异常**：`e=3` 小公钥、`e` 与 `φ(n)` 不互素、`e=1`、`e` 很大都对应不同攻击。
3. **提取参数**：`openssl rsa -pubin -in pub.pem -text -noout`。
4. **写脚本而不是手算**：所有过程都写成 py 脚本存进 `payload.md`，下次直接改参数复用。
