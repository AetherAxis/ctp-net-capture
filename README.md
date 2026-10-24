# Ctp.Net.Capture

CTP 家族的**协议取证工具**——三合一：抓包（RecordingProxy 记录代理 + 官方 SDK 驱动真实会话）、直连验收（DirectTest）、前置模板生成器（FrontGen——⑤ `ctp-net-front` 的 `CtpFrontTemplates.cs` 即本工具 `gen` 的产物，工具链关系是本仓存在的显式理由）。

依赖闭包 = 仅 ② ctp-net-core（官方 SDK wrapper，跨仓相对路径）。零 PackageReference。

## 组成

| 文件 | 职责 |
|---|---|
| `src/Ctp.Net.Capture/Program.cs` | 入口分发：抓包模式（默认）/ `gen`（模板生成）/ `tcp:` 首参（直连验收） |
| `src/Ctp.Net.Capture/RecordingProxy.cs` | 纯字节管道 TCP 记录代理（本地 17001/17002 ←→ 真实前置），逐帧落盘 hex，不解析不修改 |
| `src/Ctp.Net.Capture/DirectTest.cs` | 直连验收：握手→登录→结算→三查；支持 idle 空转（心跳/断线观测）与 proxy 录制模式（真前置 vs 本地前置线级对比） |
| `src/Ctp.Net.Capture/FrontGen.cs` | 模板生成器：抓包 S→C 应答帧 verbatim 复刻 + 标记界定字段替换；全部偏移 needle 定位 + 生成期断言（唯一命中/无重叠/余额像钱），零人工转录 |

## 用法

```bash
# 抓包（对 SimNow 真前置——五场景：故意拒单 / 激进成交 / 远端挂单 / 撤单 / idle 心跳）
dotnet run --project ctp-net-capture/src/Ctp.Net.Capture -- \
  --front tcp://180.168.146.187:10130 --mdfront tcp://180.168.146.187:10131 \
  --user <SimNow账号> --password <密码> --broker 9999

# 直连验收（本地前置或真前置——打印「✓ 登录成功」+ 三查通过；第 4 参 = idle 秒数）
dotnet run --project ctp-net-capture/src/Ctp.Net.Capture -- tcp://127.0.0.1:17801 <账号> <密码>

# 纯代理模式（只起记录代理，不驱动 SDK——供任意第三方客户端走代理打真前置，落原始帧）
#   用于 DirectTest 覆盖不到的场景：查合约（1.7 万行）、多行持仓、多行报单、行情 tick。
dotnet run --project ctp-net-capture/src/Ctp.Net.Capture -- proxy 17003 tcp://182.254.243.31:40001 /tmp/trade.txt
#   然后让客户端连 127.0.0.1:17003 即可（⑬ console：--front tcp://127.0.0.1:17003）

# 模板生成（抓包 trade.txt → CtpFrontTemplates.cs；默认输出指向兄弟仓 ctp-net-front）
dotnet run --project ctp-net-capture/src/Ctp.Net.Capture -- gen <trade.txt...> [输出.cs]
```

wire 级 bug 排查（如成交回报方向/开平槽位考证）的主场：抓真前置会话 → FrontGen 定位槽位 → 本地模板编辑 → SDK 即裁判验证。

## 红线

- **抓包原始文件（trade.txt / md.txt）含真实 SimNow 凭据与密文——永不入 git**（本仓只含工具代码；抓包产物留本地临时目录）
- 生成的 `CtpFrontTemplates.cs` 含抓包原样字节——同 ⑤ 的「永不直接公开」纪律

## 来历

2026-09-28 建于 Aether monorepo（`apps/CtpSim/Acme.Ctp.ProtoCapture`，SimNow 线协议逆向 M0 的工具）；2026-10-20 迁入 AetherAxis 并更名 `Ctp.Net.Capture`（对齐 `ctp-net-*` → `Ctp.Net.*` 命名总纲）。完整决策记录见工作区根 `MIGRATION.md`。
