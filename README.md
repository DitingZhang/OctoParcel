# Parcel Tracker

Agentic App 黑客松初赛作品。购物与物流赛道。

```text
"我的快递到哪了？"  → 被动查询器会回答这个问题
"我现在该对哪笔订单做点什么？"  → Parcel Tracker 回答这个问题
```

包裹追踪类 App 多到数不清，**它们都做错了一件事**——把"展示订单列表"当成了产品价值。但真正花你时间的从来不是"看"，是"接下来怎么办"：是去催那个停滞 3 天的卖家，还是去申请那个今晚 23:59 就关掉的退货窗口？

Parcel Tracker 做的事很简单：扫一遍订单，**挑出需要你拍板的那一笔**，把现成的话术写好，让你**确认或驳回**。它不替你下单、不替你付款、不替你发消息——它只替你判断"哪一笔最值得你花 30 秒"。

---

## 它怎么运转的

```
   启动 → 扫描 4 笔 Mock 订单
       ↓
   按确定性评分排序
       ↓
   最高分的那一笔在顶部 Banner 高亮
       ↓
   点开 → 看到建议话术
       ↓
   ┌─────────────┐
   │ 确认执行     │ → 绿色 ✓  → 记录 confirmed  → 回到列表
   │ 驳回（可选理由）│ → 灰色 ✗  → 记录 rejected  → 回到列表
   └─────────────┘
       ↓
   重新评估按钮（⟳）可以再跑一遍评分，时间流逝会改变"现在最急的是哪一笔"
```

没有任何 LLM 调用、没有外部 API。**4 笔订单是写死的，评分规则是 5 行代码**——这是初赛原型的诚实选择，把复杂度留在 UI 和状态机里，不堆在 prompt 上。

---

## 评分规则（5 行讲清楚）

| 条件 | 分数 | 标签 | 建议 |
|---|---|---|---|
| 已签收，且 7 天退货窗口**剩余不到 48 小时** | 60 | return_closing | 申请退货退款 |
| **24 小时以上没更新**物流 | 40 | stuck | 联系卖家催单 |
| 已过预计送达时间 + 未签收 | 40 | overdue | 联系卖家催单 |
| 已经过用户决策的订单 | 5 | — | 不再进 Banner |
| 其他 | 10 | normal | 灰色标签，无 Banner |

`drift = time_now() - boot_time` 反映"卡片打开至今过了多久"，所以**按 ⟳ 重新评估 时，同一笔订单可能从 80 掉到 60，从 60 掉到 5**——时间是真的在走的。

---

## 5 张截图

整条路截图存在 `bundle/screenshots/` 下：

| | |
|---|---|
| `01-overview.png` 总览<br>4 笔订单按紧急度降序，红色 Banner 锁定最高优先级 | ![overview](bundle/screenshots/01-overview.png) |
| `02-detail-stuck.png` 详情-停滞<br>Kindle 在 京东物流 36 小时没动，红色"建议立即干预"卡片给出催单话术 | ![detail-stuck](bundle/screenshots/02-detail-stuck.png) |
| `03-confirm-execute.png` 确认<br>话术再展示一遍，可填驳回理由，两颗按钮：驳回 / 确认执行 | ![confirm](bundle/screenshots/03-confirm-execute.png) |
| `04-result-ok.png` 结果-成功<br>绿色 ✓，写好待复制粘贴的文案 | ![result-ok](bundle/screenshots/04-result-ok.png) |
| `05-result-fail.png` 结果-驳回<br>灰色 ✗，记录了驳回理由，该订单回到普通优先级 | ![result-fail](bundle/screenshots/05-result-fail.png) |

绿色和灰色是**同级的两条出口**——你点驳回，结果页就是 ✗；没有任何"对不起刚才误操作"的二次挽留。这条路径也是产品的一部分，不是边缘情况。

---

## 怎么跑

```bash
cd ~/apps/script-app-demo

# 1. 校验 + 重新盖章（hub 完整性 + L0 语法）
~/agentic-new/OctoScript-App-Design-Flow/tools/octo check bundle

# 2. 启动带远程控制的 card-host（默认监听 :8142）
~/agentic-new/OctoScript-App-Design-Flow/tools/octo run bundle --port 8142 --detach

# 3. 遥控
curl -s 127.0.0.1:8142/snap                       # 控件树 JSON
curl -s '127.0.0.1:8142/click?x=200&y=345'         # 点击
curl -s 127.0.0.1:8142/quit                       # 关掉

# 或者直接截一张
~/agentic-new/OctoScript-App-Design-Flow/tools/octo shot 8142 out.png
```

需要 WSLg 或 X11 转发。`--allow-unsigned` 必须带，本次初赛构建没有发布者签名。

---

## 数据是假的，未来是真的

- 4 笔订单写在 `bundle/main.splash` 顶部，刻意覆盖三种异常路径 + 一笔正常件。
- `network.hosts = []`，`capabilities = ["storage"]`，**没有任何外发请求**。
- 状态文件 `~/apps/script-app-demo/.local-state/parcel-tracker/`：
  - `session.json` — `{boot_time}`，让"重新评估"能算出真实的时间漂移
  - `decisions.json` — `{order_id: {result, reason, action_script, decided_at}}`，避免下次启动让你对同一笔订单重新决策
- 重置：`rm -rf ~/apps/script-app-demo/.local-state/parcel-tracker`

下一步要做的事很明确——把 `orders = [...]` 换成 `fs.read("orders.json").parse_json()`，加一个 `fetch()` 拉真实物流接口。**UI、评分、状态机不动**。

---

## 已知坑

| 现象 | 原因 / 解法 |
|---|---|
| TextInput 打不出中文（WSLg 下） | `apt install fcitx5 fcitx5-chinese-addons`，`GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx`，在该环境下启动 card-host。 |
| `octo shot` / `/g` 接口返回 "grab timeout" | card-host 的 GL 帧缓冲没渲染。Linux 上：`LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=softpipe xvfb-run` 跑；或用 `MAKEPAD_WRITE_FRAMEBUFFER_PNG=/path/frame.png` 直接落盘 PNG。 |
| `octo check` 报 `publisher-signature: unsigned` | 这次没接发布者密钥。本地调试加 `--allow-unsigned`，正式上架需要 publisher key。 |
| `manifest.json` 被 `octo check` 改写 | 那是它在重新算 blake3 integrity，正常现象。 |
| curl 不到 127.0.0.1 | shell 里 `http_proxy` 走代理会劫持本地请求。加 `--noproxy '*'` 或 `NO_PROXY=127.0.0.1`。 |

---

## 技术栈

- **OctoScript (L0)** — 脚本应用语言，`bundle/main.splash` 430 行
- **OctoSense card-host** — 在能力沙箱里跑 bundle
- **Hub** — 准入与完整性校验

`bundle/` 是提交物；`.local-state/`、`target/`、日志都在外面，`.gitignore` 已经管好。

---

## 许可证

Apache-2.0。
