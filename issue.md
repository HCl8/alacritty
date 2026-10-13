# Wayland 下输入法（fcitx5 + KWin）用着用着失效

## 结论

**根因是 winit 的 bug，不是 KWin。**

winit 的 Wayland 后端在 **`wl_seat` capability 变化时**，会顺手把 `zwp_text_input_v3` 对象
`destroy()` 掉。而 text input 跟 seat capability 没有关系，只跟 seat 本身有关。对象被销毁后，
合成器会把对应的 input method context 拆掉，fcitx5 那边就会看到 `deactivate` —— 于是输入法
在已经跑起来的应用里突然失效，需要重新聚焦窗口才能恢复。

上游修复：`rust-windowing/winit@f0a83481`
（2026-10-12，*"wayland: only destroy the text input when the seat is removed"*）

> changelog 原文：*On Wayland, fix the text input being destroyed whenever a seat capability
> changed. This made IME input stop working in already-running applications after the seat's
> capabilities were updated.*

## 证据链

出问题的代码在 winit 0.30.13 的
`src/platform_impl/linux/wayland/seat/mod.rs`：

```rust
// remove_capability() —— 每次 seat capability 变化都会走到这里
if let Some(text_input) = seat_state.text_input.take() {
    text_input.destroy();          // ← 错误：不该在这里销毁
}
```

而真正该销毁的地方 `remove_seat()` 反而没管它：

```rust
fn remove_seat(...) {
    let _ = self.seats.remove(&seat.id());   // ← text_input 只是被 drop，没 destroy()
    self.on_keyboard_destroy(&seat.id());
}
```

上游的修复就是把这两处调个个儿：

```diff
  fn remove_capability(...) {
-     if let Some(text_input) = seat_state.text_input.take() {
-         text_input.destroy();
-     }
  }

  fn remove_seat(...) {
-     let _ = self.seats.remove(&seat.id());
+     if let Some(mut seat_state) = self.seats.remove(&seat.id()) {
+         if let Some(text_input) = seat_state.text_input.take() {
+             text_input.destroy();
+         }
+     }
      self.on_keyboard_destroy(&seat.id());
  }
```

## 为什么这不是 KWin 的锅

最初的分析把矛头指向 KWin：认为 KWin 在处理 `zwp_text_input_v3.done(52)` 时错误地触发了
`zwp_input_method_v1.deactivate`。日志里确实能看到 fcitx5 侧收到 `deactivate`，但**这个结论是
错的**，`done()` 的 serial 只是回执编号，跟 deactivate 没有因果关系。

正确的解释是：winit 销毁 `zwp_text_input_v3` 对象 → 合成器认为该客户端不再需要输入法 →
拆掉 input method context → fcitx5 收到 `deactivate`。**日志里的 `deactivate` 是结果，不是原因。**

社区里也有旁证：Wine 的 MR8196 提到 KWin 会对每次 `commit` 都回一个 `done`（哪怕状态没变），
被形容为 "arguably out of spec, but otherwise harmless"，Qt/GTK 的做法是**忽略这些 no-op 的
`done`**。也就是说 KWin 这点行为是已知且无害的，不该为这个 bug 负责。

## 与之无关但顺手修掉的另一个问题

alacritty 的 `update_ime_position()` 在**每帧渲染**时都会无条件发一次
`set_cursor_rectangle` + `commit()`（见上游 issue
[#8306](https://github.com/alacritty/alacritty/issues/8306) 的日志）。虽然不致命，但会造成
大量无意义的 commit。已在 `alacritty/src/display/window.rs` 里加了 rect 缓存：位置没变就不发。

## 本仓库的改动

| 仓库 | 改动 |
|---|---|
| `HCl8/winit`，分支 `local-ime-fix` | backport 上游 `f0a83481` 那 7 行到 v0.30.13 |
| `alacritty`（本仓库） | `update_ime_position()` 增加 cursor rect 缓存 |

alacritty 通过 git 依赖指向上述 fork 分支：

```toml
winit = { git = "https://github.com/HCl8/winit", branch = "local-ime-fix", default-features = false, features = ["rwh_06", "serde"] }
```

> ⚠️ 没有直接升级到 winit 0.31（beta.3 不含这个修复，而且 0.31 是一次巨大的破坏性升级，
> 上游 alacritty 自己也还停在 0.30.9）。先做最小 backport，等 0.31 正式版再说。

## 备注

- alacritty 的 `CONTRIBUTING.md` 有明确的 **no-LLM 贡献政策**，所以这些改动只留在自己的
  fork 里，不往上游提 PR。
- **尚未在真实 Wayland + fcitx5 会话里验证**，需要手动测一遍原来会失效的场景。
