# Parcel Tracker

> 主动出击的 Agentic 包裹管家——不是被动的物流查询器。

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Hackathon](https://img.shields.io/badge/Agentic_App_Hackathon-初赛-orange)](#)

## 简介

包裹追踪类 App 多到数不清，**它们都做错了一件事**——把"展示订单列表"当成了产品价值。但真正花你时间的从来不是"看"，是"接下来怎么办"：是去催那个停滞 3 天的卖家，还是去申请那个今晚 23:59 就关掉的退货窗口？

Parcel Tracker 不替你下单、不替你付款、不替你发消息——它只做一件事：**替你判断哪一笔订单最值得你花 30 秒**，把现成的话术写好，等你拍板（确认执行 / 驳回）。

Agentic App 黑客松初赛作品，购物与物流赛道。**4 笔 Mock 订单 + 确定性评分规则**，无外部 API、无 LLM 调用，能力授权只有 `storage`。

---

## 简介(项目结构)

仓库一共 15 个文件，`bundle/` 是提交物，其他都是元数据。

```
OctoParcel/
├── README.md                       ← 你正在读的
├── LICENSE                         ← Apache-2.0
├── .gitignore                      ← 忽略 .local-state/、target/、*.bak、frame*.png 等
├── AGENTS.md / CLAUDE.md / GEMINI.md   ← 通用 agent 指引（来自脚手架模板）
└── bundle/                         ← 这是会被 hub 校验和提交的目录
    ├── main.splash                 ← L0 源码 430 行（state + 18 个函数 + 3 个 pane）
    ├── manifest.json               ← id/能力/integrity
    ├── listing.json                ← App Hub 展示元数据 + 5 张截图路径
    ├── assets/
    │   └── icon.svg                ← 256×256 SVG（蓝色圆角盒 + 橙色封条）
    └── screenshots/
        ├── 01-overview.png         ← 总览（412×892）
        ├── 02-detail-stuck.png     ← 详情-停滞
        ├── 03-confirm-execute.png  ← 二次确认
        ├── 04-result-ok.png        ← 结果-确认（38 KB）
        └── 05-result-fail.png       ← 结果-驳回（灰色 ✗）
```

---

## 功能特性

- **主动扫描** — 启动时和 `⟳ 重新评估` 时重跑评分，按 `time_now() - boot_time` 算出真实时间漂移
- **三种异常识别** — 物流停滞（24 h 无更新）/ 退货窗口临近（已签收且剩余 < 48 h）/ 已逾期（过 ETA 未签收）
- **可执行的话术生成** — 给卖家催单 / 申请退货退款两类文案，写好待复制粘贴
- **授权确认流** — 详情页点"执行建议" → 二次确认卡（含驳回理由输入） → 确认才落盘，未经授权不落地
- **驳回也是合法结果** — 灰色 ✗ 路径，理由可选，订单回到正常优先级；不是边缘情况
- **跨重启记忆** — `session.json` + `decisions.json` 持久化，避免对同一笔订单反复决策
- **零外部依赖** — `network.hosts = []`，`capabilities = ["storage"]`

---

## 状态机

整个 App 围绕 `current_view`（`"overview" | "detail" | "confirm" | "result"`）和 `confirm_mode` 翻转，**没有路由库，纯全局变量 + `show()` 切换可见性**：

```
           ┌──────────────────────────────────────────┐
           │                                          │
   boot()  ▼                                          点击 "执行建议"
   ───► overview ◄────── 点击 "返回总览" ──── detail ◄─────────┐
         ▲    ▲             (from result)            │        │
         │    │                                       │        │
         │    └── 点击 Banner/卡片 ──────► detail ───┘        │
         │                                                   │
         │                                ┌──────────────────┘
         │                                │  confirm_mode = true
         │                                ▼
         │                            confirm   (同一 detail pane，下半部叠加确认卡)
         │                                │
         │   ┌────────────────────────────┼────────────────────┐
         │   │ 确认执行                    │                    │ 驳回（理由可选）
         │   ▼                            │                    ▼
         │ confirm_action()                │            reject_action()
         │   └─► result (✓ 绿)             │              └─► result (✗ 灰)
         │                                │
         │   on_click: { show("overview"); rescan() }
         └────────────────────────────────┘
```

`show(view, id)` 做的事就是：
1. 切 `current_view` / `current_order_id`
2. 切 `ui.overview` / `ui.detail` / `ui.result_pane` 三个 ScrollYView 的可见性
3. `ui.<pane>.render()` 强制 `on_render` 闭包重跑

`detail` pane 同时承载 detail 和 confirm 两种状态——靠 `confirm_mode` 这个 boolean 加一个 `if confirm_mode and s >= 40 and d == nil` 分支来判断下半部是否叠加确认卡。

---

## 评分规则（确定性，无 AI）

`fn score(o)` 5 行讲清楚：

| 条件 | 分数 | 标签 | 建议 |
|---|---|---|---|
| 已决策（`decisions[o.id] != nil`） | **5** | 已决策 | 不再进 Banner |
| 已签收 + 在 7 天退货窗口内 + **剩余不到 48 小时** | **60** | return_closing | 申请退货退款 |
| **24 小时以上没更新**物流 | **80** | 物流停滞 | 联系卖家催单 |
| 已过预计送达时间 + 未签收 | **40** | 已逾期 | 联系卖家催单 |
| 其他 | **10** | 正常 | 灰色标签，无 Banner |

实际判定顺序（`bundle/main.splash:55`）：

```splash
fn score(o){
    if decisions[o.id] != nil { return 5 }
    let h = hours_since_update(o)
    let drift = (time_now() - boot_time) / 3600.0
    if o.signed_offset_h != nil and o.signed_offset_h >= 24 and o.signed_offset_h <= 168 {
        let remaining = 168 - o.signed_offset_h - drift
        if remaining > 0 and remaining < 48 { return 60 }
    }
    if h > 24 { return 80 }
    if o.signed_offset_h == nil and o.expected_offset_h < drift { return 40 }
    return 10
}
```

`drift = (time_now() - boot_time) / 3600.0` 反映"卡片打开至今过了多久"，所以**按 `⟳ 重新评估` 时，同一笔订单可能从 80 掉到 60，从 60 掉到 5**——时间是真的在走的。

Banner 取分最高的 pending 订单。`find_pending()` 跑完就把 `pending_top` / `pending_reason` 写好给 `on_render` 用。

---

## 4 笔 Mock 订单

写在 `bundle/main.splash:12-17`，刻意覆盖三种异常路径 + 一笔正常件：

| ID | 平台 | 商品 | 承运 | 状态 | last/h | signed/h | 预期分数 |
|---|---|---|---|---|---|---|---|
| TB-2024-98765 | 淘宝 | 小米手环 8 | 顺丰 | 已到达 杭州转运中心 | 2 | nil | 10（正常） |
| JD-2024-55432 | 京东 | Kindle Paperwhite | 京东物流 | 等待自提 | 36 | nil | **80（停滞）** |
| TB-2024-12345 | 淘宝 | 罗技 MX Master 3S | 中通 | 已签收 | 36 | **144** | **60（退货窗口临近）** |
| JD-2024-77777 | 京东 | Sony WH-1000XM5 | 京东物流 | 派送中 | 22 | nil | 10 或 80（看 drift） |

启动后 `⟳` 按一次 Banner 立刻锁定 Kindle；点开 → 红色"建议立即干预" → "执行建议" → 确认 → 走 `decisions.json` 落档 → 列表里 Kindle 变成灰色"已确认"，Banner 自动切到罗技（剩余窗口 < 48 h）。

---

## 话术模板（`fn action_script(o)`）

```splash
if s == 80 or s == 40 {  // 物流停滞 或 已逾期
    "亲，我的订单 [{id}] ({name}) 在 {carrier} 已经 {floor(h)} 小时没有更新物流了，麻烦帮我催一下，谢谢！"
}
if s == 60 {  // 退货窗口临近
    "亲，我的订单 [{id}] ({name}) 已签收，想在 7 天退货窗口内申请退货退款，麻烦协助处理一下，谢谢！"
}
```

注意 `floor(h)`——`hours_since_update` 返回的是浮点（drift 累加），直接 `"" + h` 会拼出 `37.15631080104245` 这种丑陋的小数，`floor()` 把它取整成 `37`。

---

## 持久化（`fn load` / `fn save_decisions`）

启动时从 `.local-state/parcel-tracker/` 读两个 JSON：

```jsonc
// session.json
{ "version": 1, "boot_time": 1791103719.69 }   // 第一次启动写入

// decision.json
{
  "version": 1,
  "decisions": {
    "JD-2024-55432": {
      "result": "confirmed",
      "action_script": "亲，我的订单 ... 已经 36 小时没有更新物流了 ...",
      "reason": "",
      "decided_at": 1791103739.89
    }
  }
}
```

- **重置**：`rm -rf ~/apps/script-app-demo/.local-state/parcel-tracker/`
- **不会上传**：`.gitignore` 第一行就是 `.local-state/`

---

## 技术栈

- **OctoScript (L0)** — 脚本应用语言，`bundle/main.splash` 430 行 / 22647 字节
- **OctoSense card-host** — 在能力沙箱里执行 bundle（隔离文件系统、限制指令数、限制内存）
- **Hub** — 准入校验（`octo check` 干的就是这俩）
- **Wayland / X11 / 软件 GL** — card-host 的渲染后端（见"已知坑"）
- 没有任何 npm / pip / cargo 依赖

---

## 环境要求

- **`OctoScript-App-Design-Flow`** 仓库里的 `tools/octo` 和 `OctoSense-App-Hub/target/release/{card-host, hub}`（已经预编译好）
- **Linux x86_64**（已验证 Ubuntu 24.04 WSL2）
- **WSLg 或本机 X11** — card-host 需要 Wayland / X 上下文
- **可选** Xvfb — headless 截图测试用
- **Git + 已配 SSH key** — GitHub 推送（仓库是 `git@github.com:DitingZhang/OctoParcel.git`）

中文输入法 fcitx5 装好了但**没法在 card-host 里打中文**——`octo run` 包装器没把 `GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx` 三个变量透传到 card-host 进程（`/proc/<pid>/environ` 验证为空），且 sandbox 里 dbus 没启、`fcitx5-remote` 直接抛异常。本次提交**不依赖中文 IME**。

---

## 安装与运行

```bash
# 1. 拉代码
git clone git@github.com:DitingZhang/OctoParcel.git
cd OctoParcel

# 2. 校验 + 重新盖章（hub blake3 + L0 语法）
~/agentic-new/OctoScript-App-Design-Flow/tools/octo check bundle
# 期望: parcel-tracker 0.1.0 — PASSED

# 3. 启动带远程控制的 card-host
~/agentic-new/OctoScript-App-Design-Flow/tools/octo run bundle --port 8142 --detach
# 期望: listening on 127.0.0.1:8142, first frame drawn

# 4. 验证
curl --noproxy '*' -s 127.0.0.1:8142/s | python3 -m json.tool | head

# 5. 跑一遍流程
curl --noproxy '*' -s '127.0.0.1:8142/click?x=200&y=345'   # 点 Kindle 卡
curl --noproxy '*' -s '127.0.0.1:8142/click?x=206&y=522'   # 点 执行建议
curl --noproxy '*' -s '127.0.0.1:8142/click?x=296&y=870'   # 点 确认执行

# 6. 截图（任选其一）
~/agentic-new/OctoScript-App-Design-Flow/tools/octo shot 8142 out.png
# 或带环境变量直接落盘：
MAKEPAD_WRITE_FRAMEBUFFER_PNG=/tmp/frame.png \
  ~/agentic-new/OctoSense-App-Hub/target/release/card-host \
    --bundle ~/apps/script-app-demo/bundle \
    --app-data ~/apps/script-app-demo/.local-state \
    --allow-unsigned --stamp
```

`--allow-unsigned` 必须加——本次没接 publisher key。

---

## 流程截图

| | |
|---|---|
| `01-overview.png` 总览<br>4 笔订单按紧急度降序，红色 Banner 锁定最高优先级 | ![overview](bundle/screenshots/01-overview.png) |
| `02-detail-stuck.png` 详情-停滞<br>Kindle 在 京东物流 36 小时没动，红色"建议立即干预"卡片给出催单话术 | ![detail-stuck](bundle/screenshots/02-detail-stuck.png) |
| `03-confirm-execute.png` 确认<br>话术再展示一遍，可填驳回理由，两颗按钮：驳回 / 确认执行 | ![confirm](bundle/screenshots/03-confirm-execute.png) |
| `04-result-ok.png` 结果-成功<br>绿色 ✓，写好待复制粘贴的文案 | ![result-ok](bundle/screenshots/04-result-ok.png) |
| `05-result-fail.png` 结果-驳回<br>灰色 ✗，记录了驳回理由，该订单回到普通优先级 | ![result-fail](bundle/screenshots/05-result-fail.png) |

绿色和灰色是**同级的两条出口**——你点驳回，结果页就是 ✗；没有"刚才误操作了吗"的二次挽留。这条路径也是产品的一部分，不是边缘情况。

---

## 已知坑

| 现象 | 根因 / 解法 |
|---|---|
| `/g` 接口返 `{"err":"grab timeout"}` | card-host 的 GL 帧缓冲没渲染（Wayland 后端 + WSLg 不会画 frame） |
| 想在 Linux 上抓图 | `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=softpipe xvfb-run` 起 card-host；或 `MAKEPAD_WRITE_FRAMEBUFFER_PNG=/path/frame.png` 直接落盘 PNG（绕过 `/g`） |
| `octo check` 报 `publisher-signature: unsigned` | 没接发布者密钥；本地调试加 `--allow-unsigned`，上架要 publisher key |
| `manifest.json` 被 `octo check` 改写 | 它在重新算 `integrity.bundle_blake3`，**正常现象**，每次改 `bundle/` 都该重跑 |
| `curl 127.0.0.1` 不通 | shell 里 `http_proxy=172.21.144.1:10810` 劫持了本地请求，加 `--noproxy '*'` 或 `NO_PROXY=127.0.0.1` |
| TextInput 打不出中文 | 本环境 fcitx5 没真的能工作（dbus 缺失 + env 没透传），不在本次提交承诺范围内 |
| `on_render` 闭包里调 `action_script(...)` 不显示结果 | L0 闭包对函数返回值处理有 quirk，必须先 `let script_txt = action_script(o)` 再 `text: script_txt` |
| `Label{draw_bg +:...}` 不支持 | Label 没有 draw_bg，要包一层 `SolidView` 再嵌 Label |
| 点不到按钮 | `/click?x=N&y=M` 用控件树里 `r=[x,y,w,h]` 的中心点；按钮不一定在视觉位置 |

---

## 后续工作

下一步要做的事很明确——把 `orders = [...]` 换成 `fs.read("orders.json").parse_json()`，加一个 `fetch()` 拉真实物流接口。**UI、评分、状态机不动**。再往后的功能升级（自动发消息、自动退款）需要接 host 服务，超出本次提交范围。

---

## 许可证

Apache-2.0，详见 `LICENSE`。
