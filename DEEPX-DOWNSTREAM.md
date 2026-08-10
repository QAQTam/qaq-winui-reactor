# DEEPX-DOWNSTREAM — reactor fork 补丁登记

本仓库是 windows-reactor 的 **durable fork**：主仓库 `F:\DeepX` 以
`path` 依赖（开发态）/ git rev（发布态）引用，所有下游扩展以补丁形式
提交在本 fork 历史中。本文件登记**与上游的差异**，供合并上游 / 重评
补丁时核对。

> 登记以「本轮已核实」为准；历史补丁以 `git log --oneline` 为完整来源，
> 下表只列登记时点仍生效的关键差异。

## 当前引用

- 主仓库 `apps/winui/Cargo.toml`：
  `windows-reactor = { path = "../../../deepx-winui-reactor/crates/libs/reactor" }`
- fork 基线 HEAD（登记时点）：`614cf8688`

## 补丁登记表

| # | 补丁 | 文件 | 说明 | 引入 commit |
|---|------|------|------|-------------|
| P1 | DEEPX_PERF_LOG 慢渲染日志 | `crates/libs/reactor/src/engine.rs` | 环境变量门控（`DEEPX_PERF_LOG=<path>`）；单次渲染 >3ms 时追加写一行（pid / render# / tree / reconcile / effects / diffed / skipped / created）。零依赖零分配（未设 env 时仅一次 var 查询）。用于定位持续高 CPU 的成本构成。 | `614cf8688` |
| P2 | `set_render_observer` 每帧渲染观察者 | `crates/libs/reactor/src/engine.rs` | thread_local 全局槽，`RenderCompleteInfo`（tree/reconcile/effects ms + diff/skipped/created）每帧回调；`set_render_observer(Some/None)` 注册/注销。UI 线程专用（跨线程注册静默指向该线程槽）；回调内禁止再 set（RefCell 重入）。apps/winui `diagnostics.rs` 消费。 | `614cf8688` |

## 备注

- `RenderStats` / `RenderCompleteInfo` / `stats()` 为 fork 既有公开 API
  （`engine.rs` L1105+），P2 复用同一 payload 结构。
- fork 历史补丁（on_frame / 段落 diff / 修饰方法 / 滚动修复等）见
  `git log --oneline --all`，主仓库侧对应文档见 `docs/reactor-winui3-knowledge.md`
  与 `docs/windows-reactor-skill.md`。
- 合并上游前逐条核对：P1/P2 均为**纯增量**（无行为变更），与上游
  reconcile 路径正交。
