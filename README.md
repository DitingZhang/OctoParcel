# OctoParcel

> Agentic 包裹管家——主动出击,不是被动查询。

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Hackathon](https://img.shields.io/badge/Agentic_App_Hackathon-2026-orange)](https://github.com/gosimfoundation/hackathon-agenticapp26)
[![Track](https://img.shields.io/badge/Track-购物_与物流-success)](#)
[![Built with](https://img.shields.io/badge/Built_with-OctoScript_×_AutoSense-lightgrey)](https://github.com/OctoSense-org/OctoScript-App-Design-Flow)

## 这是什么应用

OctoParcel 是一个跑在 OctoSense AutoSense OS 上的 OctoScript bundle。

它替你盯着购物订单——主动挑出**需要你拍板的那一笔**(物流停滞、退货窗口即将关闭、超过预计送达时间未签收),把现成的话术写好,让你**确认执行或驳回**。

- 不替你下单、不替你付款、不替你发消息
- 不展示订单列表就完事——它替你判断"哪一笔最值得你花 30 秒"
- 4 笔 Mock 订单 + 确定性评分规则,无外部 API、无 LLM 调用
- 能力授权只有 `storage`(只写本地 JSON,不外发任何请求)

## 怎么跑起来

```bash
# 1. 拉代码
git clone https://github.com/DitingZhang/OctoParcel.git
cd OctoParcel

# 2. 校验 bundle
~/agentic-new/OctoScript-App-Design-Flow/tools/octo check bundle

# 3. 启动(需要 X11 / WSLg)
~/agentic-new/OctoScript-App-Design-Flow/tools/octo run bundle --port 8142 --detach

# 4. 验证
curl --noproxy '*' -s 127.0.0.1:8142/snap | python3 -m json.tool | head
```

打开 App Hub GUI 或 `card-host` 窗口,即可看到界面。

## 验证情况

### 已验证
- ✅ `octo check bundle` → PASSED
- ✅ card-host 启动 → `first frame drawn`
- ✅ 4 笔 Mock 订单按紧急度降序
- ✅ 点 Kindle 卡 → 详情页 → 催单话术
- ✅ 执行建议 → 确认页 → 确认执行 → 结果页(✓ 绿)
- ✅ 返回 → 重新评估 → Banner 切换到罗技
- ✅ 执行建议 → 驳回(理由)→ 结果页(✗ 灰)
- ✅ 持久化:重启后 `decisions.json` 保留
- ✅ 5 张截图覆盖完整流程

### 未验证
- ⏳ 真实物流 API(当前是 Mock 数据)
- ⏳ 中文输入(环境限制:fcitx5 在 card-host 里不工作)
- ⏳ 手机端(等 App Hub 上架)

## Agentic 体现在哪

不是聊天机器人,也不是"通知中心"。三件事让它算 Agentic:

1. **主动监控**——启动后扫描所有订单 + 顶部"⟳ 重新评估"按钮按需重扫。按真实时间漂移计算。
2. **生成可执行方案**——按订单状态生成具体话术(给卖家催单 / 申请退货退款),写好待复制粘贴。
3. **授权闭环**——话术只是预览,**未经确认不落地**。"执行建议" → 二次确认卡 → "确认执行" 才写入持久化。驳回也是合法结果,记录理由,订单回到普通优先级。

整条链路是**观察 → 建议 → 授权 → 落档**。用户始终是决策者。

## 截图长什么样

### 总览

4 笔 Mock 订单按紧急度降序排列。顶部红色 Banner 锁定最高优先级的那一笔。每张卡左侧色条 + 右侧标签表示严重程度。

![总览](bundle/screenshots/01-overview.png)

### 详情-停滞

Kindle 在 京东物流 36 小时没动,详情页红色"⚠ 建议立即干预"卡片给出催单话术预览,下面是"执行建议"按钮。

![详情-停滞](bundle/screenshots/02-detail-stuck.png)

### 二次确认

话术再展示一遍,下方一个可选的驳回理由输入框,两颗按钮:**驳回** 或 **确认执行**。

![二次确认](bundle/screenshots/03-confirm-execute.png)

### 结果-成功

绿色 ✓ + "已生成行动预案" + 待复制粘贴到卖家聊天窗口的文案 + "返回总览" 按钮。

![结果-成功](bundle/screenshots/04-result-ok.png)

### 结果-驳回

灰色 ✗ + "已记录驳回决定" + 写入了驳回理由,订单回到正常优先级。

![结果-驳回](bundle/screenshots/05-result-fail.png)

绿色和灰色是**同级的两条出口**——你点驳回,结果页就是 ✗,没有"刚才误操作了吗"的二次挽留。

## Demo Video

[1 分 30 秒完整流程演示](视频链接)
(视频链接我稍后补,先填 TODO 占位。)


## 技术栈

- **OctoScript (L0)** — 脚本应用语言,`bundle/main.splash` 430 行
- **OctoSense card-host** — 在能力沙箱里执行 bundle
- **Hub** — 准入与完整性校验
- **Makepad** — GPU 渲染引擎

## 数据来源与限制

| 项 | 当前状态 |
|---|---|
| 订单数据 | 4 笔 Mock 订单(演示用,覆盖 3 类异常 + 1 笔正常) |
| 物流状态 | 确定性规则模拟(按 `last_update` / `signed_offset` 计算) |
| API 调用 | 无(`capabilities = ["storage"]`, `network.hosts = []`) |
| LLM 调用 | 无(评分是 5 行确定性规则,可验证、无幻觉风险) |
| 后续计划 | 接 APIZero 真实快递接口(复赛阶段) |

## 仓库结构

```
OctoParcel/
├── README.md                       ← 你正在读的
├── LICENSE                         ← Apache-2.0
├── bundle/                         ← 提交到 App Hub 的目录
│   ├── main.splash                 ← L0 源码
│   ├── manifest.json               ← id/能力/integrity
│   ├── listing.json                ← App Hub 展示元数据
│   ├── assets/icon.svg
│   └── screenshots/                ← 5 张流程截图
├── docs/
│   ├── architecture.md             ← 状态机 / 评分规则 / 数据流
│   └── known-issues.md             ← 调试期发现的环境问题
└── build/                          ← .gitignored,本地产物
    ├── SUBMISSION.md               ← App Hub issue 文案
    └── review.json                 ← hub scan packet
```

## 许可证

Apache-2.0,详见 [`LICENSE`](LICENSE)。
