# OctoParcel — Architecture

本文档面向开发者,描述内部实现。README 是用户视角,这是代码视角。

## 状态机

整个 App 围绕 `current_view` 和 `confirm_mode` 两个全局变量翻转。没有路由库,纯 `show()` 切换 ScrollYView 可见性:

```
   boot()
   ───► overview ◄─── "返回总览" ──── detail ◄─── 点击订单卡 / Banner
         ▲                                │
         │                                │ confirm_mode = true
         │                                ▼
         │                            confirm (detail pane 下半部叠加确认卡)
         │                                │
         │   ┌────────────────────────────┼──────────────────┐
         │   │ 确认执行                    │                  │ 驳回 (理由可选)
         │   ▼                            │                  ▼
         │ confirm_action()              │          reject_action()
         │   └─► result (✓ 绿)           │             └─► result (✗ 灰)
         │                                │
         │   on_click: { show("overview"); rescan() }
         └────────────────────────────────┘
```

`show(view, id)` 三件事:
1. 切 `current_view` / `current_order_id`
2. 切三个 ScrollYView 的可见性 (`overview` / `detail` / `result_pane`)
3. `ui.<pane>.render()` 强制 on_render 闭包重跑

`detail` pane 同时承载 detail 和 confirm 两种状态——靠 `confirm_mode` boolean + `if confirm_mode and s >= 40 and d == nil` 分支判断下半部是否叠加确认卡。

## 评分规则

`fn score(o)` 5 行讲清楚:

| 条件 | 分数 | 标签 | 建议 |
|---|---|---|---|
| 已决策 (`decisions[o.id] != nil`) | **5** | 已决策 | 不再进 Banner |
| 已签收 + 在 7 天退货窗口内 + **剩余不到 48 小时** | **60** | return_closing | 申请退货退款 |
| **24 小时以上没更新**物流 | **80** | 物流停滞 | 联系卖家催单 |
| 已过预计送达时间 + 未签收 | **40** | 已逾期 | 联系卖家催单 |
| 其他 | **10** | 正常 | 灰色标签,无 Banner |

判定顺序 (`bundle/main.splash:55`):

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

`drift = (time_now() - boot_time) / 3600.0` 反映"卡片打开至今过了多久"——按 `⟳ 重新评估` 时同一笔订单可能从 80 掉到 60,从 60 掉到 5。

## 4 笔 Mock 订单

`bundle/main.splash:12-17`,覆盖三种异常路径 + 一笔正常件:

| ID | 平台 | 商品 | 承运 | last/h | signed/h | 预期分数 |
|---|---|---|---|---|---|---|
| TB-2024-98765 | 淘宝 | 小米手环 8 | 顺丰 | 2 | nil | 10 (正常) |
| JD-2024-55432 | 京东 | Kindle Paperwhite | 京东物流 | 36 | nil | **80 (停滞)** |
| TB-2024-12345 | 淘宝 | 罗技 MX Master 3S | 中通 | 36 | **144** | **60 (退货窗口临近)** |
| JD-2024-77777 | 京东 | Sony WH-1000XM5 | 京东物流 | 22 | nil | 10 或 80 (看 drift) |

## 话术模板

`fn action_script(o)` (`main.splash:83`):

```splash
if s == 80 or s == 40 {  // 物流停滞 或 已逾期
    "亲,我的订单 [{id}] ({name}) 在 {carrier} 已经 {floor(h)} 小时没有更新物流了,麻烦帮我催一下,谢谢!"
}
if s == 60 {  // 退货窗口临近
    "亲,我的订单 [{id}] ({name}) 已签收,想在 7 天退货窗口内申请退货退款,麻烦协助处理一下,谢谢!"
}
```

注意 `floor(h)`——`hours_since_update` 返回的是浮点 (`drift` 累加),直接 `"" + h` 会拼出 `37.15631080104245` 这种丑陋小数,`floor()` 取整成 `37`。

## 持久化

启动时从 `.local-state/parcel-tracker/` 读两个 JSON:

```jsonc
// session.json
{ "version": 1, "boot_time": 1791103719.69 }   // 第一次启动写入

// decisions.json
{
  "version": 1,
  "decisions": {
    "JD-2024-55432": {
      "result": "confirmed",
      "action_script": "亲,我的订单 ... 已经 36 小时没有更新物流了 ...",
      "reason": "",
      "decided_at": 1791103739.89
    }
  }
}
```

- 重置: `rm -rf ~/apps/script-app-demo/.local-state/parcel-tracker/`
- 不会上传: `.gitignore` 第一行就是 `.local-state/`

## 函数清单 (`main.splash`)

| 行号 | 函数 | 作用 |
|---|---|---|
| 31 | `load()` | 从 session.json / decisions.json 读持久化状态 |
| 46 | `save_decisions()` | 写 decisions.json |
| 50 | `hours_since_update(o)` | 计算当前时间漂移后的"自上次更新到现在"小时数 |
| 55 | `score(o)` | 评分,见上表 |
| 68 | `urgency_text(s)` | 分数 → 标签 |
| 76 | `action_text(s)` | 分数 → 建议动作 |
| 83 | `action_script(o)` | 分数 + 订单 → 完整话术 |
| 95 | `sort_orders()` | 按 score 降序排序(L0 没内置 sort,用选择排序手写) |
| 114 | `find_pending()` | 找 Banner,让一批优先级最高那笔 |
| 143 | `find_order(id)` | 按 id 查订单 |
| 150 | `show(view, id)` | 切换 pane 可见性 + 切全局变量 |
| 163 | `rescan()` | `find_pending()` + 重渲染 overview |
| 168 | `confirm_action()` | 用户点确认:写入 decisions,跳 result |
| 185 | `reject_action()` | 用户点驳回:写入 decisions,跳 result |
| 203 | `boot()` | 启动序列:`load` → `find_pending` → `sort_orders` → `show(overview)` |

`start_timeout(0.1, || boot())` 在 body 求值完毕后触发启动。
