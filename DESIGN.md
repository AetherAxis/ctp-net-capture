# Ctp.Net.Capture 设计说明

> 本文件写「怎么设计 / 为什么这么设计 / 还能往哪升级」。工作区边界与迁移决策以根 `MIGRATION.md` 为准。

## 1. 定位与边界

CTP 家族的协议取证工具，三合一：抓包（记录代理 + 官方 SDK 驱动真实会话）、直连验收（不落盘，只验链路）、模板生成（抓包 S→C 帧 → ⑤ 的 `CtpFrontTemplates.cs`）。

只产证据与模板，不被引用。依赖闭包仅 ② `ctp-net-core` 一个 `ProjectReference`，零 `PackageReference`。

## 2. 项目组成

| 文件 | 职责 | 关键类型 | 出向依赖 |
|---|---|---|---|
| `Program.cs` | 入口分发、抓包编排 | 顶层语句、`Args()` | core 的 `Trader`/`MarketData`、本仓另三类 |
| `RecordingProxy.cs` | 双向转发、逐帧落盘 hex | `RecordingProxy` | 仅 BCL |
| `DirectTest.cs` | 握手→登录→结算→三查；idle/soak/proxy | `DirectTest` | `Trader`、`RecordingProxy` |
| `FrontGen.cs` | 抓包解析 → needle 定位 → 生成模板 | `FrontGen` | 仅 BCL |

`FrontGen` 不引用 core：生成是纯字节处理。

## 3. 关键设计决策

### 3.1 记录代理是纯字节管道

- **决策**：只做 `ReadAsync`→`WriteAsync`→追加 `[时间戳] #连接 C→S/S→C [N bytes] hex`，解析留给离线。
- **理由**：不污染证据——解析即假设，错一次毁掉整次取证；代理挂死还会拖垮 SDK 会话。
- **代价**：记录单位是 TCP 读块而非协议帧，`FrontGen` 须按长度再切。

### 3.2 零人工转录 + 等宽纪律

- **决策**：偏移全由 needle 长度与命中位置计算，源码无手数常量；生成前断言唯一命中、编辑区不重叠、余额「像钱」、⑯ 六槽转义解码等宽同值，失败即非零退出。整帧 verbatim，只替换等宽标记字段（`F(s,width)`）与请求侧已证可变宽的字段（价格/量/TradeId）。
- **理由**：wire 错位不报错而是让 SDK 静默失步——宁可响亮失败；变宽会致记录遍历失步。
- **代价**：断言与抓包强绑（`BalHex`、m2701、18 帧交替），换账号即失败；失真边界无回归检测。

### 3.3 工具链单向：`gen` → ⑤ 的模板文件

- **决策**：默认输出 ⑤ 的 `src/Ctp.Net.Front/CtpFrontTemplates.cs`，默认抓包取 `$TMP/ctp-proto-capture` 下帧数最多的 `trade.txt`。
- **理由**：模板是抓包原样字节，属证据；分仓让 ⑤ 不被临时目录绑住。
- **代价**：产物入库而生成器在另一仓，漂移无人发现（6.1/6.2）。

### 3.4 三种模式靠首参分发

- **决策**：首参 `gen` → `FrontGen`；`tcp:` 前缀 → `DirectTest`；其余 → 抓包模式。
- **理由**：位置参数与选项天然互斥，首参守卫即可，无需 CLI 框架。
- **代价**：无 `--help`，非法选项静默回退默认值。

## 4. 运行与验证

```bash
# 抓包 / 直连（[idle] [proxy] [soak]）/ 模板生成
dotnet run --project src/Ctp.Net.Capture -- --front tcp://<host>:10130 --mdfront tcp://<host>:10131 --user <账号> --password <密码> --broker 9999
dotnet run --project src/Ctp.Net.Capture -- tcp://<host> <账号> <密码>
dotnet run --project src/Ctp.Net.Capture -- gen <trade.txt...> [输出.cs]
```

验证是 E2E：`DirectTest` 打出「登录成功 + 三查通过」即链路成立；`FrontGen` 的断言即生成期验收；产物由 ⑤ 的 E2E（SDK 即裁判）裁决。CI 只跑 Release 构建。

## 5. 升级记录（2026-10-24）

本轮**代码零变更**：调查开始时 `git status --short` 与 `git diff` 均为空，HEAD 停在 `2d5fe1c`；源码与 CI 均未改，唯一新增是本文件（会话期间 `.gitignore` 另有一条非本任务改动）。本仓是纯工具，静态重构无法在无凭据环境验证；且零包依赖、零内部 API 面，无需跟随升级的契约。

## 6. 待升级项

| # | 优先级 | 项目 | 现状证据 | 建议 | 代价 / 风险 |
|---|---|---|---|---|---|
| 1 | P0 | 无守护测试 | 1 个 csproj、0 个测试项目；ci.yml 明写「不伪造测试状态」 | 固定样本跑 `gen`，断言产物可编译且与基线一致 | 引入测试包破零依赖；样本须脱敏 |
| 2 | P0 | 生成器与产物已漂移 | ⑤ 产物含 `EscapedGbkFixed` 与「2026-10-13 手改批」，`FrontGen` 只发 `EscapedGbk`，且注明「原始抓包已清空」 | 把手改回灌进 `FrontGen`，用 #1 的 diff 锁定 | 需重建样本 |
| 3 | P1 | 生成不可在 CI 复现 | M2/M3 帧来自 v1 拒绝对、v2 推送、v3 多会话，需真前置凭据 | 裁剪必要帧为脱敏样本入仓 | 触及凭据红线 |
| 4 | P1 | 断言与抓包强绑 | `BalHex`、余额 6 命中、`ExtractDay` 等均硬编码 | 余额按 E0 转义结构扫描 8 逻辑字节自推导 | 报错精度下降 |
| 5 | P2 | 端口固定不可并行 | `Program.cs` 写死 17001/17002，`DirectTest` proxy 写死 17001 | 端口可参数化或动态分配 | 需同步 README |
