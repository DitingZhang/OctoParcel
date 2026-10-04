# Parcel Tracker

> An Agentic parcel butler for your shopping & logistics — proactive, not passive.

Parcel Tracker is a submission to the **Agentic App Hackathon** (初赛 / Track: Shopping & Logistics). It watches your mock orders, **flags the ones that need your decision** (stuck in transit, return window closing, overdue), proposes an action script you can paste into the seller chat, and lets you **confirm or reject with one tap**. It is the opposite of a status-tracker: it does not answer "where is my package?" — it decides what you should do about it.

---

## Why this is Agentic

A parcel tracker only shows you a list. A butler does something with it.

| Capability | Where it shows up |
|---|---|
| **Proactive monitoring** of mock data on every boot and on demand (`⟳ 重新评估`) | Banner at the top of the overview, red urgency tag on each card |
| **Generates a concrete action** (e.g. a polite nudge to the seller) | Red "建议立即干预" card on the detail page |
| **Asks for your authorization** before any script is finalized | "执行建议" button → confirm card with full preview |
| **Honors a rejected decision** and downgrades the order back to normal priority | Gray "✗" result page; persisted in `decisions.json` |
| **Persists state across restarts** so you do not re-decide the same order every launch | `session.json` + `decisions.json` in `.local-state/` |

The flow is **Observe → Suggest → Authorize → Record**. The user stays in charge; nothing happens without a tap.

---

## Flow (5 screenshots)

| # | Screenshot | What is happening |
|---|---|---|
| 01 | ![Overview](bundle/screenshots/01-overview.png) | Overview: 4 mock orders sorted by urgency. The red banner picks the top pending one (Kindle, stuck 36 h). The colored bar + tag on each card show severity. |
| 02 | ![Detail — stuck](bundle/screenshots/02-detail-stuck.png) | Detail of the stuck Kindle order. Red "建议立即干预" card with the full action script preview. |
| 03 | ![Confirm — execute](bundle/screenshots/03-confirm-execute.png) | Confirm card. The same script is shown one more time, an optional reject-reason field, and two buttons: **驳回** or **确认执行**. |
| 04 | ![Result — ok](bundle/screenshots/04-result-ok.png) | After **确认执行**: green ✓, "已生成行动预案", the script to copy into the seller chat. |
| 05 | ![Result — fail](bundle/screenshots/05-result-fail.png) | After **驳回**: gray ✗, "已记录驳回决定", reason recorded, order drops back to normal priority. |

This pair of result pages (green / gray) is the **failure-state requirement** of the hackathon — Parcel Tracker can produce either outcome depending on user choice, and both are first-class.

---

## Scoring rules (deterministic, no LLM)

```
fn score(o):
    drift = time_now() - boot_time      // hours since the card opened

    if o.signed_offset_h ∈ [24, 168]      // already signed, but the 7-day return
       and remaining_hours = 168 - o.signed_offset_h - drift
       and 0 < remaining_hours < 48:     // window closes in < 2 days
        → 60  return_closing  · "申请退货退款"

    if last_update_h - drift > 24:       // no movement for > 24 h
        → 80  stuck           · "联系卖家催单"

    if o.signed_offset_h == nil
       and o.expected_offset_h < drift:  // past ETA, not signed
        → 40  overdue         · "联系卖家催单"

    if decided (confirmed / rejected):   // score from before; user took action
        → 5   (excluded from banner)

    else:
        → 10  normal          (no banner, gray tag)
```

The highest-scoring pending order drives the banner. The "stuck" and "overdue" branches generate the same nudge template; the "return_closing" branch generates a different one.

---

## How to run

```bash
# from this directory
cd ~/apps/script-app-demo

# 1. lint + stamp the bundle
~/agentic-new/OctoScript-App-Design-Flow/tools/octo check bundle

# 2. launch the remote viewer
~/agentic-new/OctoScript-App-Design-Flow/tools/octo run bundle --port 8142 --detach

# 3. drive it
curl -s 127.0.0.1:8142/snap          # widget tree as JSON
curl -s 127.0.0.1:8142/click?x=200&y=345   # click a card
curl -s 127.0.0.1:8142/quit          # stop

# or take a screenshot directly
~/agentic-new/OctoScript-App-Design-Flow/tools/octo shot 8142 out.png
```

Run on a host with WSLg or X11 forwarding. `--allow-unsigned` is required because the hackathon build has no publisher signature yet.

---

## Data and persistence

- **Mock data only.** Four hard-coded orders are baked into `bundle/main.splash` (top of the file). They cover all three urgency paths (stuck, return-closing, overdue) and one normal case.
- No real APIs, no network calls. The `network.hosts` list in `bundle/manifest.json` is empty. `capabilities = ["storage"]` is the only grant.
- Persistence lives in `~/apps/script-app-demo/.local-state/parcel-tracker/`:
  - `session.json` — `{version: 1, boot_time: <seconds>}` so the "重新评估" button can show real time elapsed.
  - `decisions.json` — `{version: 1, decisions: {order_id: {result, reason, action_script, decided_at}}}` so a decided order stays decided across restarts.
- To start fresh: `rm -rf ~/apps/script-app-demo/.local-state/parcel-tracker`.

---

## Known limits

- **Mock data, not a real carrier.** The four orders are baked in. A future revision would replace `orders = [...]` at the top of `bundle/main.splash` with `fs.read("orders.json").parse_json()` plus a `fetch()` host call — the UI and scoring do not change.
- **Action script is a string, not an API call.** The result page tells the user to paste it. Driving fulfillment out of the app would need a host service (e.g. `taobao.send_message`) and is out of scope for this submission.
- **Chinese IME requires `fcitx5`.** On WSLg, install `fcitx5 fcitx5-chinese-addons`, set `GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx XMODIFIERS=@im=fcitx`, then launch card-host under that environment. ASCII-only input works without any of this.
- **Screenshots require a windowed or virtual display.** card-host uses GL; headless mode needs `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=softpipe xvfb-run` on Linux, or `MAKEPAD_WRITE_FRAMEBUFFER_PNG=/tmp/frame.png` for direct PNG output (bypasses `/g`).
- **No publisher signature.** `octo check` accepts `--allow-unsigned` for local testing; publishing to the App Hub requires a publisher key (out of scope here).

---

## Stack

- **OctoScript** (L0) — the script-app language. See `OctoScript-App-Design-Flow/docs/SCRIPT-API.md`.
- **OctoSense card-host** — the runtime that executes the bundle under a capability jail.
- **Hub** — admission + integrity check.

Everything in `bundle/` is the submitted artifact. Everything else (`.local-state/`, `target/`, logs) stays outside.

---

## License

Apache-2.0. See `LICENSE`.
