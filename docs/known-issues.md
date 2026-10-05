# Known Issues (开发期环境问题)

本节记录开发过程中遇到的环境问题,不是产品 bug。用户视角的 README 不提这些。

| 现象 | 根因 | 解法 |
|---|---|---|
| `curl 127.0.0.1:8142` 不通 | shell 里 `http_proxy=172.21.144.1:10810` 劫持了本地请求 | 加 `--noproxy '*'` 或 `NO_PROXY=127.0.0.1` |
| `/g` 接口返 `{"err":"grab timeout"}` | card-host 的 GL 帧缓冲没渲染 (Wayland + WSLg 不会画 frame) | `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=softpipe xvfb-run`;或 `MAKEPAD_WRITE_FRAMEBUFFER_PNG=/path/frame.png` 直接落盘 |
| TextInput 打不出中文 | fcitx5 装好了但 sandbox 里 dbus 没启,`fcitx5-remote` 抛异常;且 `octo run` 没把 `GTK_IM_MODULE=fcitx` 等三个变量透传给 card-host 进程 (`/proc/<pid>/environ` 验证为空) | 不在本次提交承诺范围内;ASCII 输入不受影响 |
| 点不到按钮 | `/click?x=N&y=M` 需要用控件树里 `r=[x,y,w,h]` 的中心点;视觉位置不可靠 | 先 `curl /snap` 取 `r=`,算中心,再 `/click` |
| `bundle/screenshots/01-main.png` 是占位 | 之前是模板生成的占位图,后来替换成 5 张新截图 | 已删除占位,listing.json 已更新 |
| `manifest.json` 被 `octo check` 改写 | hub stamp 重新算 `integrity.bundle_blake3` | 正常现象,每次改 `bundle/` 都该重跑 |
| `octo check` 报 `publisher-signature: unsigned` | 没接发布者密钥 | 本地调试加 `--allow-unsigned`;上架要 publisher key |

## L0 语法怪癖 (开发期遇到,不是产品问题)

| 写法 | 行为 |
|---|---|
| `Label{... draw_bg +:{...} ...}` | Label 没有 draw_bg,编译期错误 "field draw_bg not found in type-check"。要包一层 `SolidView` |
| `text: action_script(o)` 在 on_render 闭包里 | 闭包对函数返回值的处理有 quirk,渲染出来是空格。先 `let s = action_script(o)` 再 `text: s` 即可 |
| `text: result_script` 全局变量在 on_render 闭包里读 | 同上 quirk,如果右值是 fn_call 必须先 `let s = ...` |
| `let x = fn_call(...)` | 顶层 `let` 重赋值支持,但 `global = fn_call(...)` 在某些路径下会被当成比较表达式。先 `let local = ...` 再 `global = local` |

## 后续工作

按 OctoLoop 迭代步骤("先完成再完美"):

1. ✅ 基本功能 (扫描 + 评分 + 详情 + 确认 + 结果 + 持久化)
2. ⏳ 接 OctoSense 系统 agent 上的 agentic 功能 (目前是脚本层面的 agentic,系统级待办)
3. ⏳ UI 完善 (后续 round)

下一步要做的事很明确——把 `orders = [...]` 换成 `fs.read("orders.json").parse_json()`,加一个 `fetch()` 拉真实物流接口。**UI、评分、状态机不动**。再往后的功能升级 (自动发消息、自动退款) 需要接 host 服务,超出本次提交范围。
