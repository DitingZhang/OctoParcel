# Parcel Tracker（包裹管家）

> 主动出击的 Agentic 包裹管家——不是被动的物流查询器。

Parcel Tracker 是 **Agentic App 黑客松初赛**（购物与物流赛道）的参赛作品。它持续监控你的订单，**主动挑出需要决策的包裹**（物流停滞、退货窗口临近、已逾期），为每笔异常生成可一键复制粘贴的行动文案（催单话术/退货申请），并让你通过一次点击**确认执行或驳回**。它回答的不是"我的包裹到哪儿了"，而是"**我现在该对哪个包裹做点什么**"。

---

## 为什么这是 Agentic

普通包裹追踪器只展示列表，管家会代你判断。

| 能力 | 在哪里体现 |
|---|---|
| **主动监控**：启动时扫描 + "⟳ 重新评估" 按钮按需扫描 | 总览顶部红色 Banner，每张订单卡左侧色条和右侧紧急度标签 |
| **生成具体行动方案**（如礼貌的催单话术） | 详情页红色 "⚠ 建议立即干预" 卡片，附完整文案预览 |
| **未经授权不落地**：所有动作前等你拍板 | "执行建议" 按钮 → 确认页再次预览 → "确认执行" 才落盘 |
| **驳回也是合法结果**：被拒绝的订单回到正常优先级，结果页有专门的"✗"路径 | 灰色 "✗" 结果页，写入 `decisions.json` |
| **跨重启保持记忆**：不会让你对同一笔订单反复决策 | `session.json` + `decisions.json` 在 `.local-state/` 下持久化 |

整条链路是**观察 → 建议 → 授权 → 落档**。用户始终是决策者，没有一次点击就什么也不会发生。

---

## 完整流程（5 张截图）

| # | 截图 | 发生了什么 |
|---|---|---|
| 01 | ![总览](bundle/screenshots/01-overview.png) | 总览：4 笔 Mock 订单按紧急度降序排列。红色 Banner 锁定最高优先级的一笔（Kindle，已停滞 36 小时）。每张卡片左侧色条和右侧标签表示严重程度。 |
| 02 | ![详情-停滞](bundle/screenshots/02-detail-stuck.png) | Kindle 订单详情。红色 "⚠ 建议立即干预" 卡片给出完整催单话术预览，下面是红色 "执行建议" 按钮。 |
| 03 | ![确认-执行](bundle/screenshots/03-confirm-execute.png) | 二次确认页。话术再展示一遍，下方一个可选的驳回理由输入框，两颗按钮：**驳回** 或 **确认执行**。 |
| 04 | ![结果-成功](bundle/screenshots/04-result-ok.png) | 点 **确认执行** 后：绿色 ✓ + "已生成行动预案" + 待复制粘贴到卖家聊天窗口的文案 + "返回总览" 按钮。 |
| 05 | ![结果-驳回](bundle/screenshots/05-result-fail.png) | 点 **驳回** 后：灰色 ✗ + "已记录驳回决定" + 写入了驳回理由，订单回到正常优先级。 |

这一对结果页（绿色 / 灰色）就是黑客松要求的**失败状态**——Parcel Tracker 既能产出成功也能产出失败，两条路径都是一等公民。

---

## 评分规则（确定性，无 AI）

```
fn score(o):
    drift = time_now() - boot_time      // 卡片打开至今过了多少小时

    if o.signed_offset_h ∈ [24, 168]    // 已签收，但在 7 天退货窗口内
       and remaining = 168 - o.signed_offset_h - drift
       and 0 < remaining < 48:         // 窗口不到 2 天就要关
        → 60  return_closing  · "申请退货退款"

    if last_update_h - drift > 24:      // 24 小时以上没动过
        → 80  stuck           · "联系卖家催单"

    if o.signed_offset_h == nil
       and o.expected_offset_h < drift: // 超过预计送达时间且未签收
        → 40  overdue         · "联系卖家催单"

    if 已决策（confirmed / rejected）：   // 已经过用户拍板
        → 5   （不再进 Banner）

    else:
        → 10  normal          （灰色标签，无 Banner）
```

最高分的待决策订单驱动顶部 Banner。stuck / overdue 走同一套催单话术，return_closing 走另一套退货话术。

---

## 怎么跑起来

```bash
# 切到本目录
cd ~/apps/script-app-demo

# 1. lint + 重新盖章
~/agentic-new/OctoScript-App-Design-Flow/tools/octo check bundle

# 2. 启动带远程控制的 card-host
~/agentic-new/OctoScript-App-Design-Flow/tools/octo run bundle --port 8142 --detach

# 3. 遥控它
curl -s 127.0.0.1:8142/snap                       # 取出控件树 JSON
curl -s 127.0.0.1:8142/click?x=200\&y=345         # 点击一张卡片
curl -s 127.0.0.1:8142/quit                       # 关掉

# 或者直接截图
~/agentic-new/OctoScript-App-Design-Flow/tools/octo shot 8142 out.png
```

环境需要 WSLg 或 X11 转发。`--allow-unsigned` 必须加上，因为这次黑客松构建没有发布者签名。

---

## 数据来源与持久化

- **纯 Mock 数据**。4 笔订单硬编码在 `bundle/main.splash` 顶部，覆盖三种异常路径（停滞、退货窗口、逾期）+ 一笔正常件。
- **没有真实 API，没有网络请求**。`bundle/manifest.json` 中 `network.hosts` 为空，`capabilities = ["storage"]` 是唯一授权。
- 持久化目录 `~/apps/script-app-demo/.local-state/parcel-tracker/`：
  - `session.json` — `{version: 1, boot_time: <秒>}`，让 "重新评估" 按钮反映真实时间流逝。
  - `decisions.json` — `{version: 1, decisions: {order_id: {result, reason, action_script, decided_at}}}`，保证同一笔订单不会让你反复决策。
- 重置： `rm -rf ~/apps/script-app-demo/.local-state/parcel-tracker`。

---

## 已知限制

- **是 Mock，不是真实快递**。订单是写死的。未来版本会把 `bundle/main.splash` 顶部的 `orders = [...]` 换成 `fs.read("orders.json").parse_json()` + `fetch()`，但 UI 和评分无需改动。
- **行动文案只是字符串，不是 API 调用**。结果页告诉用户去粘贴。真正的自动履行需要 host 服务（如 `taobao.send_message`），超出本次提交范围。
- **中文输入需要 fcitx5**。WSLg 下安装 `fcitx5 fcitx5-chinese-addons`，设 `GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx`，再用该环境启动 card-host。纯英文输入不需要任何额外配置。
- **截图需要图形环境**。card-host 用 GL，纯命令行环境需要 `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=softpipe xvfb-run`，或用 `MAKEPAD_WRITE_FRAMEBUFFER_PNG=/tmp/frame.png` 直接落盘 PNG（绕过 `/g` 接口）。
- **没有发布者签名**。`octo check --allow-unsigned` 本地测试通过；正式上架 App Hub 需要发布者密钥（不在本次提交范围内）。

---

## 技术栈

- **OctoScript (L0)** — 脚本应用语言。详见 `OctoScript-App-Design-Flow/docs/SCRIPT-API.md`。
- **OctoSense card-host** — 在能力沙箱里执行 bundle 的运行时。
- **Hub** — 准入与完整性校验。

`bundle/` 是提交物，其他（`.local-state/`、`target/`、日志）都在外面。

---

## 许可证

Apache-2.0。详见 `LICENSE`。
