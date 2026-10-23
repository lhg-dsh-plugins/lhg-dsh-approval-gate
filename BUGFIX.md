# BUGFIX — v0.5.3

## Bug 标题

`permissionPresets.current(session.events)` 传参错误：审批门控完全静默失效，审批历史恒为 0

## 证据链

- 上游缺陷报告（同一根因，均针对已发布版本且在上游 HEAD 仍未修复）：
  - issue #3 / #15 / #17 / #21 / #25：`permissionPresets.current()` 期望 **session 对象**——宿主实现 `current(session)` → `permissionState(session)` → `sessionProjections.stateOf(session, "permissions")` → `materializeCells(session)`，内部迭代 `session.events`。插件在 `src/index.mjs` 的 `approval/request` 处理器中传入的是 `session.events`（数组），宿主在数组上取 `.events` 得到 `undefined`，迭代时抛 `TypeError: events is not iterable`。
  - 异常被处理器自身的 try/catch 吞掉并 `return next()`，于是：无论会话选中哪个预设（含 `auto-approve`），判定管道永远走不到；`events.jsonl` / `audit.log` 从不生成，UI「审批」视图恒为空。
- 本地离线复现（修复前，exit 1）：按宿主语义实现 `permissionPresets` mock（`stateOf` 迭代 `session.events`），以临时 `DSH_HOME` 加载真实插件并驱动其 `approval/request` 处理器，发送一条选中 `auto-approve` 预设、模式为 `workspace-write`（命中默认白名单、应自动放行）的请求：
  - 实际：`permissionPresets.current failed TypeError: events is not iterable`（抛出点 `src/index.mjs:1290`），处理器返回 `next()` 的结果 `forwarded-to-human`（next 被调用 1 次），`events.jsonl` 未生成、审计无 ALLOW 记录。
  - `git stash` 回退修复后复跑，结果逐字一致（FAIL）。

## 根因

`src/index.mjs` 中 `permissionPresets.current(session.events)` 把事件数组误当作 session 对象传入。宿主 API 的入参是 session 对象本身，事件迭代由宿主内部完成。

## 改动文件

- `src/index.mjs`：1 行 —— `permissionPresets.current(session.events)` → `permissionPresets.current(session)`。
- `package.json`：version `0.5.2` → `0.5.3`；`repository.url` 改指本 fork 仓库。
- `README.md`：顶部追加 fork 说明头（上游原文未删改）。
- 新增 `UPSTREAM.md`、`BUGFIX.md`。
- 其余文件（含 `LICENSE`、`client.js`、`cordis.patch.yml`）与上游逐字一致。上游无 lock 文件、无独立版本戳常量，无待同步项。

## 复验结果

- 同一复现脚本（修复后，exit 0）：处理器返回 `allowed-once`，`next()` 未被调用，`events.jsonl` 已生成自动放行事件，`audit.log` 含 ALLOW 记录。
- stash 对照：回退修复 → 复现 FAIL；恢复修复 → 复现 PASS。
- `npm test`（上游唯一检查项：`node --check src/index.mjs && node --check client.js`）修复前后均通过。

## 验证边界

- 未在真实 DSH 宿主（0.1.5-rc.2 / 0.2.x）中端到端运行；宿主 `current(session)` 的签名依据上游 issue #21/#25 中对 `@deepseek-ai/dsh-permission-presets` 源码的核对记录，mock 按该语义实现。
- issue #25 提到的第二处问题（`PROFILE_PATCH_PATH` 硬编码 `web` profile）不在本版修复范围，本版只修上述一处传参 bug。
