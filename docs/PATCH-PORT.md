# DeepX 补丁移植手册（旧 fork → 新 fork）

> 状态：快照已落地（commit `021ef1b0e`），**语义融合尚未完成**，当前分支大概率编译不过。
> 本手册是唯一移植依据：按分组逐项融合，每步跑门禁，最后更新登记表。

## 1. 背景

| 项 | 值 |
|---|---|
| 旧 fork（补丁来源） | `F:\deepx-winui-reactor-1`，分支 `deepx-reactor`，HEAD `de10e3642` |
| 旧基线 | 上游 `9e4eb04e4a`（#4807，2026-08-08） |
| 新 fork（当前） | `F:\deepx-winui-reactor`，分支 `deepx-winui`，HEAD `021ef1b0e` |
| 新基线 | 上游 `5d7a5d8890`（#4814，2026-08-10，含 #4808/#4811/#4812/#4813/#4814） |
| 上游在途 | PR #4815 "centralize child projection ownership"（open，**合入前先别做 reconciler 层移植**） |
| 迁移方式 | 文件级快照（非 commit 移植）；`reconciler.rs` 的 +11 行已合入 `reconciler/mod.rs` |

快照 commit 内容：29 文件、+4936/-268。

## 2. 补丁分组与风险评级

| 组 | 文件 | 行数 | 风险 | 说明 |
|---|---|---|---|---|
| A 引擎/诊断 | `engine.rs`(+82)、`lib.rs`(+18)、`hooks.rs` | 104 | 🟢 低 | P1 `DEEPX_PERF_LOG`、P2 `set_render_observer`、`on_frame`；上游未动这些区域 |
| B 后端 WinUI | `backend/mod.rs`(+26)、`backend/winui/mod.rs`(+940)、`convert.rs`(+48)、`generated_attach_event.rs` | 1016 | 🟠 中 | DeepX 的 WinUI 后端扩展（RichTextBlock、templated scroll、渐变 brush 等）；需与上游 #4795（image/text trimming）逐段对比 |
| C 绑定生成物 | `bindings.rs`(+1453)、`reactor_selftest/src/bindings.rs`(+1430)、`tools/reactor/src/base.txt`(+36) | 2919 | 🟠 中 | **优先用生成工具重新生成**（`cargo run -p reactor -- update` 之类）；上游 #4801/#4794 改过 bindgen |
| D reconciler 层 | `reconciler/mod.rs`(+11，已合)、`reconciler/templated.rs`(+76) | 87 | 🔴 高 | 与上游重构系列正面重叠；`templated.rs` 是唯一"确定冲突点" |
| E widgets | `widget.rs`(+166)、`button.rs`(+65)、`flyout.rs`(+50)、`text_block.rs`(+50)、`image.rs`(+14)、`tab_view.rs`(+9)、`composition_host.rs`、`swap_chain_panel.rs` | 360 | 🟠 中 | DeepX 扩展 widget（动画 translation、flyout def、渐变）；上游 #4795 动过 image/文本 |
| F 修饰系统 | `style.rs`(+66)、`element.rs`(+105) | 171 | 🟢 低 | `GradientBrush`、`foreground_gradient`、modifiers 扩展；上游未动 |
| G 测试 | `tests/downstream_features.rs`(+174，新增)、`tests/src/lib.rs`(+48)、`image_bindings.rs`、`rich_text_builder.rs` | 243 | 🟢 低 | 纯新增/独立 |
| H 文档 | `DEEPX-DOWNSTREAM.md`、`docs/workflow.md` | 113 | 🟢 低 | 登记表需更新基线 |

## 3. 已知冲突点（融合时逐个处理）

### 3.1 `reconciler/templated.rs` — 唯一确定冲突 🔴
- 旧补丁在 `#[cfg(debug_assertions)] self.debug_assert_native_ownership();` 之后插入 `scroll_pending` 重试循环
- 上游 #4811 把这行改成了 `self.assert_consistent_inner();`（并新增统一自检）
- 融合：**保留上游的 `assert_consistent_inner()`**，`scroll_pending` 重试代码插在其后；验证 scroll 重试路径跑完自检不误报
- 同时确认访问路径：`self.tree.templated.lists` 在 #4812/#4814 拆分后是否仍成立（`MountedTree` 移入 `mounted_tree.rs`，字段未改名，大概率成立）

### 3.2 `reconciler/mod.rs` — 已合入，需验证
- 快照已把 `foreground_gradient` 两处插入 `mod.rs`（`diff_modifiers` / `diff_element`）
- 验证点：`PropValue::Gradient` 在 backend 的 set_prop 路径是否完整（`backend/winui/mod.rs` 补丁里有实现）；`mods.foreground_gradient` 字段来自 `style.rs`（已拷贝）

### 3.3 `bindings.rs` — 生成物 vs 手工补丁
- 上游 #4801/#4794 改 `windows-bindgen` 生成逻辑；补丁的 bindings 是旧生成 + 手工加（RichTextBlock 相关）
- 融合：先跑生成工具得到新 bindings，再叠加手工差异；若生成工具流程不通，退化为手动合并

### 3.4 `backend/winui/mod.rs` — 逐段对比
- 上游 #4795 改过 image icon sizing / text trimming / component state，可能与补丁的后端扩展重叠
- 融合：`git diff` 三方对比（基线 / 上游新 / 补丁版），逐 hunk 决定

### 3.5 上游在途 PR #4815
- "centralize child projection ownership"（+374/-172，7 文件，`r6` 分支）——**若近期合入，D 组要等它落地后重做**
- 建议：D 组（reconciler 层）最后做，等 #4815 状态明朗

## 4. 移植步骤（顺序执行）

```powershell
# 0) 前置：确认上游状态
gh api repos/microsoft/windows-rs/pulls/4815 --jq .state   # 若 merged，先 fetch 合入再开始

# 1) 编译探底：拿到当前快照的真实错误清单（预期集中在 B/C/D/E）
cargo check -p windows-reactor 2>&1 | Select-String "^error" | Group-Object { ($_ -split "-->")[1] } | Select Name,Count

# 2) A 组 + F 组 + G 组 + H 组（低风险，先融合）——预期零冲突
#    验证：cargo check -p windows-reactor

# 3) E 组 widgets：与上游新版本逐文件对比融合
git diff 9e4eb04e4a 5d7a5d8890 -- crates/libs/reactor/src/widgets/   # 上游在 widgets 的改动
#    验证：cargo check

# 4) B 组 backend：逐段融合（重点 image/text trimming 区域）
#    验证：cargo check

# 5) C 组 bindings：优先重新生成，退路是手动合并
#    验证：cargo check

# 6) D 组 reconciler（最后）：templated.rs scroll 补丁 + mod.rs 渐变验证
#    验证：cargo test -p windows-reactor --tests

# 7) 全量门禁
cargo check --workspace
cargo test -p windows-reactor
cargo test -p reactor_selftest
git diff --check

# 8) 更新 DEEPX-DOWNSTREAM.md：基线改 5d7a5d8890、登记表按实际融合结果重写、标注 #4815 状态
```

## 5. 完成后必做

- [ ] 更新 `DEEPX-DOWNSTREAM.md`（新基线、补丁清单、冲突解决记录）
- [ ] 更新 `F:\DeepX\scripts\dev-downstream.ps1`：`$Rev` → 新 fork 快照 commit，`$Local` 对齐实际路径
- [ ] 在 `F:\DeepX` 跑 `pwsh scripts/dev-downstream.ps1 off` 确认 git rev 依赖可用
- [ ] 通知 DeepX 侧：`apps/winui`、`markdown-winui`、`deepx-fluent` 消费面无 API 变化（补丁未动公开 API 形状）
