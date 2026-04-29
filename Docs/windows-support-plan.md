# Remodex Windows 适配计划书

## Context

用户的 fork 位于 `/Users/liuhan/Desktop/test/remodex`。当前 Remodex 的核心 bridge 是 Node.js，`remodex up` / `remodex run` 在非 macOS 平台已经会走前台 `startBridge()`，但完整体验仍明显偏 macOS：后台常驻依赖 `launchd`，桌面刷新依赖 AppleScript / `osascript`，桌面操作依赖 `open` / `pgrep` / `pkill` / `caffeinate`，设备状态同步依赖 macOS Keychain `security` 命令，启动脚本也只有 Bash 版本。

目标是把 Windows 支持拆成可交付、可验证的阶段：先让 Windows 用户能稳定前台运行 bridge 并通过 iPhone 控制 Codex，再逐步补齐安全状态存储、git/path 兼容、桌面集成、后台服务和 CI。整个过程必须保持 local-first，不引入托管服务假设，不硬编码远程域名，并避免破坏现有 macOS 行为。

## Recommended Approach

采用“平台适配层 + 分阶段交付”的方式，而不是在业务代码中散落大量 `process.platform` 判断。

优先级顺序：

1. Windows 前台 bridge MVP：`remodex up/run` 可用。
2. Windows 本地状态、git、项目路径兼容。
3. Windows Codex 桌面集成与 thread resume。
4. Windows 后台常驻服务与 `start/status/stop/restart` 命令。
5. Windows CI、文档、安装脚本和端到端验证。

## Execution Standards

### 平台抽象规范

- 新增或提取平台适配工具，集中处理平台差异。
- 不在核心业务逻辑中反复写 `process.platform === "win32"`。
- 平台专属行为放进小模块：命令执行、URL 打开、后台服务、桌面 App、凭据存储、空设备路径、wake lock。
- 不支持的 Windows 功能必须返回明确、可操作的错误信息，不要静默失败。

### 安全规范

- 保持 local-first：不新增默认远程 relay 或 hosted-service 假设。
- 不记录 relay `sessionId`、pairing token 等 bearer-like 信息。
- Windows 设备状态和 pairing 信息不能降级为宽权限明文存储。
- Git、workspace、desktop 操作必须继续使用现有 permission / approval 语义。

### 代码质量规范

- 优先复用现有状态读写工具：`phodex-bridge/src/daemon-state.js`。
- 优先复用现有 bridge 启动逻辑：`phodex-bridge/src/bridge.js` 的 `startBridge()`。
- 优先复用现有 Codex transport Windows 分支：`phodex-bridge/src/codex-transport.js` 的 `createCodexLaunchPlans()`。
- 不做大范围重命名，除非能明显降低平台混乱。
- 现有 macOS 行为必须保持兼容，已有测试要继续通过。

### 测试规范

- 使用现有 `node:test` 风格。
- Windows 服务、Credential Manager、进程管理等 OS 操作必须 mock，不要求 CI 具备管理员权限。
- 平台相关测试通过依赖注入模拟 `platform`、`spawn`、`execFile`、`fs`、`os`。
- Windows 路径测试必须覆盖空格、反斜杠、drive letter。

## Requirement Breakdown

## Phase 1 — Windows Foreground MVP

目标：Windows 上可前台启动 bridge，iPhone 可通过 relay/QR 连接并控制本机 Codex。

### Scope

- 保留 `remodex up` / `remodex run` 前台模式。
- 确保 Windows 不调用 macOS-only 命令。
- 抽象 auth URL 打开逻辑。
- 抽象 wake lock，Windows 先允许 no-op。
- 验证 `codex app-server` Windows 启动逻辑。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/bin/remodex.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/bridge.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/codex-transport.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/index.js`

### Reuse

- 复用 `startBridge()`：`phodex-bridge/src/bridge.js`。
- 复用 `createCodexLaunchPlans()` 中已有 `win32` 分支：`phodex-bridge/src/codex-transport.js`。
- 复用 CLI 的 `emitResult()` / `emitJson()` / `emitVersion()`：`phodex-bridge/bin/remodex.js`。

### Implementation Checklist

- [ ] 新增平台工具模块，例如 `phodex-bridge/src/platform.js`。
- [ ] 新增命令执行工具，例如 `phodex-bridge/src/command-runner.js`。
- [ ] 新增 URL 打开工具，例如 `phodex-bridge/src/open-url.js`。
- [ ] 将 `bridge.js` 中 `openPendingAuthLoginOnMac` 泛化为跨平台 opener。
- [ ] 将 `createMacOSBridgeWakeAssertion` 抽象为 `wake-lock`，Windows MVP 可 no-op 并记录状态。
- [ ] 确认 Windows 前台 `remodex up/run` 不触发 `launchctl`、`open`、`osascript`、`caffeinate`。
- [ ] 给 `remodex-cli.test.js` 增加 Windows 前台命令路由测试。

### Acceptance Criteria

- Windows 上 `remodex up` / `remodex run` 能进入前台 bridge 路径。
- Windows 上不会尝试执行 macOS-only 命令。
- auth URL 可用 Windows 默认浏览器打开；打开失败时打印手动访问 URL。
- macOS 原有 `up` 启动 launchd 服务的行为不变。

## Phase 2 — Shared Platform Utilities

目标：建立后续适配所需的统一平台基础设施。

### Critical Files

新增建议：

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/platform.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/command-runner.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/null-device.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/open-url.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/wake-lock.js`

### Implementation Checklist

- [ ] `platform.js` 提供 `isMacOS()`、`isWindows()`、`isLinux()`、`platformName()`。
- [ ] `command-runner.js` 封装 `spawn` / `execFile`，默认不用 shell，支持注入 runner。
- [ ] `null-device.js` 提供 `nullDevicePath()`：Windows 返回 `NUL`，Unix 返回 `/dev/null`。
- [ ] `open-url.js` 统一打开 URL、路径、应用协议。
- [ ] `wake-lock.js` 将 macOS `caffeinate` 逻辑移出 `bridge.js`。
- [ ] 新增对应工具测试，模拟 macOS / Windows / Linux。

### Acceptance Criteria

- 后续功能不直接依赖 `/bin/sh`、`open`、`osascript`、`launchctl`、`caffeinate`。
- 平台差异有集中测试。

## Phase 3 — Device State and Pairing Safety

目标：Windows 上 pairing/device state 可安全读写，`reset-pairing` 可用。

### Current Findings

- `phodex-bridge/src/secure-device-state.js` 使用 macOS Keychain `security` 命令。
- `phodex-bridge/src/daemon-state.js` 使用 `~/.remodex` 和 `mode: 0o600`，并已 best-effort `chmodSync()`。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/secure-device-state.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/daemon-state.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/bin/remodex.js`

### Reuse

- 继续复用 `resolveRemodexStateDir()`、`writeDaemonConfig()`、`writePairingSession()`、`writeBridgeStatus()` 等状态工具：`daemon-state.js`。
- 继续复用 `resetBridgeDeviceState()` 暴露给 CLI 的路径：`src/index.js`。

### Implementation Checklist

- [ ] 新增凭据后端抽象，例如 `phodex-bridge/src/credential-store.js`。
- [ ] macOS backend 保持现有 Keychain `security` 行为。
- [ ] Windows backend 使用 Windows Credential Manager 或经过验证的安全凭据库。
- [ ] 把 POSIX `chmod 600` 包装成权限工具，Windows 不依赖 POSIX mode。
- [ ] 确保 `remodex reset-pairing` 在 Windows 删除对应 device state / credential state。
- [ ] 添加 Windows credential backend mock 测试。

### Acceptance Criteria

- Windows pairing state 不会因为调用 macOS `security` 崩溃。
- Windows 不把敏感配对状态降级为宽权限明文。
- `reset-pairing` 在 Windows 可用并有清晰成功/失败输出。

## Phase 4 — Git and Workspace Path Compatibility

目标：Windows 上 git 操作、workspace patch/revert、路径展示可靠。

### Current Findings

- `phodex-bridge/src/git-handler.js` 大体可移植，但 `gitDiffNoIndexNumstat()` / `gitDiffNoIndexPatch()` 使用 `/dev/null`。
- `phodex-bridge/src/project-handler.js` quick locations 偏 Mac。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/git-handler.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/workspace-handler.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/project-handler.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/null-device.js`

### Implementation Checklist

- [ ] 将 `git-handler.js` 中 `/dev/null` 替换为 `nullDevicePath()`。
- [ ] 验证 Git for Windows 对 `NUL` 与 `/dev/null` 的实际行为，采用 Node `spawn` 下最稳定方案。
- [ ] 增加带空格路径、Windows drive letter、反斜杠路径测试。
- [ ] 为 `project-handler.js` 增加 Windows quick locations：Desktop、Documents、Downloads、source、repos、dev。
- [ ] quick locations 返回前校验路径是否存在，保持输出格式稳定。

### Acceptance Criteria

- `git/status`、`git/diff`、`workspace/revertPatchPreview`、`workspace/revertPatchApply` 在 Windows 路径下可测。
- Windows 用户看到合理的项目快捷目录。
- macOS 现有路径建议不回退。

## Phase 5 — Desktop Integration on Windows

目标：Windows 上支持打开/恢复 Codex 桌面端；刷新能力先定义清晰 fallback。

### Current Findings

- `phodex-bridge/src/desktop-handler.js` 直接拒绝非 `darwin`，并依赖 `open` / `pgrep` / `pkill` / `caffeinate`。
- `phodex-bridge/src/codex-desktop-refresher.js` 依赖 `/bin/sh -lc`、`osascript`、`/Applications/Codex.app`。
- `phodex-bridge/src/session-state.js` 使用 `open -b`。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/desktop-handler.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/codex-desktop-refresher.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/session-state.js`

新增建议：

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/desktop/index.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/desktop/macos.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/desktop/windows.js`

### Implementation Checklist

- [ ] 把 macOS 桌面操作移动到 macOS desktop adapter，保留现有行为。
- [ ] Windows desktop adapter 支持显式环境变量指定 Codex Desktop 路径，例如 `REMODEX_CODEX_DESKTOP_PATH`。
- [ ] Windows app discovery 查找常见安装位置，但不要硬编码唯一位置。
- [ ] `session-state.js` 改用通用 opener，不直接 `open -b`。
- [ ] `codex-desktop-refresher.js` 将 AppleScript 刷新抽象为平台 adapter。
- [ ] Windows 刷新若没有可靠机制，返回 structured unsupported result，并提示仍可通过 iPhone live stream 使用。
- [ ] 所有进程发现/启动/停止命令通过 command runner mock 测试。

### Acceptance Criteria

- Windows 上不再因 `desktop/*` 请求直接报“only macOS”。
- Windows 可打开 Codex Desktop 或给出明确设置路径指引。
- Windows 刷新不可用时不影响 iPhone live stream。
- macOS AppleScript refresh 保持兼容。

## Phase 6 — Windows Background Service

目标：Windows 支持 `remodex start/status/stop/restart`，达到接近 macOS launchd 的后台常驻体验。

### Current Findings

- macOS 后台实现集中在 `phodex-bridge/src/macos-launch-agent.js`。
- CLI 当前通过 `assertMacOSCommand()` 阻止非 macOS 使用 service 命令。
- 后台状态文件由 `daemon-state.js` 统一处理，可复用。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/bin/remodex.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/index.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/macos-launch-agent.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/daemon-state.js`

新增建议：

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/daemon-controller.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/src/windows-service.js`

### Recommended Windows Service Strategy

优先采用“用户级 Task Scheduler”而不是机器级 Windows Service，因为当前 macOS `LaunchAgent` 也是用户级常驻模型，且通常不需要管理员权限。

### Implementation Checklist

- [ ] 把 CLI 的 service 操作抽象为 daemon controller：`start()`、`stop()`、`restart()`、`status()`、`runService()`。
- [ ] macOS controller 复用 `macos-launch-agent.js`。
- [ ] Windows controller 使用 Task Scheduler 或等价用户级机制。
- [ ] Windows service 运行 Node CLI 的 service entrypoint，例如 `remodex run-service` 或新增 `run-windows-service`。
- [ ] `run-service` 不能无条件调用 macOS backend，应按平台分发。
- [ ] status 输出统一包含：platform、installed、running、pid/status、daemonConfig、bridgeStatus、pairingSession、log paths。
- [ ] Windows 日志路径继续使用 `daemon-state.js` 的 logs 目录。
- [ ] CLI JSON 输出保持稳定。

### Acceptance Criteria

- Windows 上 `remodex start` 能启动后台 bridge 或给出权限/配置错误。
- Windows 上 `remodex status --json` 能输出机器可读状态。
- Windows 上 `remodex stop/restart` 可用。
- 单元测试不创建真实 scheduled task；只验证命令构造和状态解析。
- macOS `start/status/stop/restart` 行为不变。

## Phase 7 — Scripts, Packaging, and Docs

目标：Windows 用户有清晰安装/启动路径。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/package.json`
- `/Users/liuhan/Desktop/test/remodex/run-local-remodex.sh`
- `/Users/liuhan/Desktop/test/remodex/README.md`
- `/Users/liuhan/Desktop/test/remodex/Docs/self-hosting.md`

新增建议：

- `/Users/liuhan/Desktop/test/remodex/run-local-remodex.ps1`

### Implementation Checklist

- [ ] 确认 `package.json` 的 `bin.remodex` 可通过 npm 在 Windows 生成 shim。
- [ ] 尽量新增跨平台 npm script，减少维护 PowerShell/Bash 双脚本。
- [ ] 如果需要本地 relay + bridge 一键启动，新增 `run-local-remodex.ps1`。
- [ ] Windows 文档说明 Node、Git for Windows、Codex CLI、relay、Tailscale 推荐方式。
- [ ] 文档明确：前台模式先可用，后台服务和桌面刷新能力按版本说明。
- [ ] 不加入硬编码远程域名或 hosted-service 默认值。

### Acceptance Criteria

- Windows 用户可以按 README 在本机启动 relay/bridge。
- 文档解释 foreground 与 background 模式差异。
- 文档包含常见失败排查：PATH、Git、Codex、浏览器打开、Task Scheduler 权限、desktop path。

## Phase 8 — CI and Verification Hardening

目标：跨平台回归保护。

### Critical Files

- `/Users/liuhan/Desktop/test/remodex/.github/workflows/bridge-check.yml`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/test/remodex-cli.test.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/test/macos-launch-agent.test.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/test/daemon-state.test.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/test/codex-desktop-refresher.test.js`
- `/Users/liuhan/Desktop/test/remodex/phodex-bridge/test/git-handler.test.js`

### Implementation Checklist

- [ ] GitHub Actions 增加 `windows-latest` 和 `macos-latest` matrix。
- [ ] 保持 privileged Windows service 测试为 mock 或手动 opt-in。
- [ ] 增加 Windows 平台工具测试。
- [ ] 增加 CLI Windows command routing 测试。
- [ ] 增加 Codex transport Windows launch plan 测试。
- [ ] 增加 Windows git/path 测试。
- [ ] 增加 Windows credential mock 测试。
- [ ] 增加 Windows desktop adapter 测试。

### Acceptance Criteria

- Linux/macOS/Windows CI 均能跑 bridge 单元测试。
- CI 不需要安装真实 Codex Desktop。
- CI 不需要管理员权限。
- macOS launch-agent 测试通过 mock 保持稳定。

## Execution Checklist by Milestone

### Milestone 1: Windows Foreground MVP

- [ ] 加平台工具、command runner、open-url、wake-lock。
- [ ] 改 `bridge.js`，移除 Windows 路径上的 macOS 命令调用。
- [ ] 保持 `remodex up/run` 前台运行。
- [ ] 增加 Windows CLI 路由测试。
- [ ] 本地运行 `cd phodex-bridge && npm test`。

### Milestone 2: Windows State + Git + Paths

- [ ] 加 credential-store 抽象。
- [ ] 实现 Windows credential backend。
- [ ] 适配 `secure-device-state.js`。
- [ ] 适配 `git-handler.js` 的 null device。
- [ ] 增加 Windows quick locations。
- [ ] 增加状态/git/path 测试。

### Milestone 3: Windows Desktop Integration

- [ ] 拆分 desktop handler adapter。
- [ ] Windows 支持打开 Codex Desktop 或 explicit path。
- [ ] `session-state.js` 改用通用 opener。
- [ ] `codex-desktop-refresher.js` 拆分平台刷新策略。
- [ ] 添加桌面相关 mock 测试。

### Milestone 4: Windows Background Service

- [ ] 加 daemon-controller 抽象。
- [ ] 保持 macOS launch agent backend。
- [ ] 新增 Windows Task Scheduler backend。
- [ ] 改 CLI `start/status/stop/restart/run-service` 平台分发。
- [ ] 增加 Windows service mock 测试。

### Milestone 5: Docs + CI + Release Readiness

- [ ] 增加 Windows CI matrix。
- [ ] 增加 Windows 本地运行文档。
- [ ] 如有必要新增 `run-local-remodex.ps1`。
- [ ] 补充 troubleshooting。
- [ ] 做 Windows 真机 smoke test。

## Verification Plan

### Automated Verification

在 `/Users/liuhan/Desktop/test/remodex/phodex-bridge`：

```bash
npm test
```

在 `/Users/liuhan/Desktop/test/remodex/relay`：

```bash
npm test
```

CI 目标：

- `ubuntu-latest`
- `macos-latest`
- `windows-latest`

### Windows Manual Smoke Test

在 Windows 真机或 VM：

1. 安装 Node.js、Git for Windows、Codex CLI。
2. 确认 `node`、`npm`、`git`、`codex` 在 PATH。
3. 启动 relay：`cd relay && npm install && npm start`。
4. 启动前台 bridge：`cd phodex-bridge && npm install && npm start` 或 `remodex up`。
5. 用 iPhone 扫 QR 配对。
6. 发送普通 prompt，确认 Codex 输出实时流回 iPhone。
7. 测试 auth URL 打开。
8. 测试 `git/status`、`git/branches`、`git/log`。
9. 在带空格路径的 repo 下测试 git diff/status。
10. 测试 `remodex reset-pairing`。
11. 如果实现后台服务，测试 `remodex start/status/stop/restart`。
12. 如果实现桌面集成，测试 resume / open Codex Desktop。

### Regression Verification

macOS 上必须确认：

- `remodex up` 仍使用 launchd 后台服务并打印 QR。
- `remodex start/status/stop/restart` 行为不变。
- AppleScript refresh 逻辑不回退。
- pairing state 与 Keychain 逻辑不回退。

## Risks and Mitigations

### Risk: Windows 后台服务需要权限

Mitigation：优先用户级 Task Scheduler；失败时提示使用前台 `remodex up/run`。

### Risk: Windows credential 实现引入 native dependency

Mitigation：先评估依赖安装稳定性；通过 `credential-store` 抽象隔离实现，必要时可替换。

### Risk: Windows 路径和 shell quoting 出错

Mitigation：尽量使用 `spawn` / `execFile` 参数数组；路径含空格纳入测试。

### Risk: Codex Desktop Windows 安装路径不稳定

Mitigation：支持显式 `REMODEX_CODEX_DESKTOP_PATH`；自动发现只作为辅助。

### Risk: macOS 行为回退

Mitigation：macOS 逻辑先移动到 adapter，不重写；添加 macOS CI；保持现有测试。

## Definition of Done

Windows 支持可认为完成，当满足：

- Windows 前台 bridge 能完成 QR 配对、发送 prompt、接收 Codex 流式输出。
- Windows 本地 pairing/device state 可安全持久化和重置。
- Windows git/status/diff/path 基本操作通过测试。
- Windows 用户有清晰文档可启动 self-host/local relay。
- CI 至少覆盖 Linux/macOS/Windows 的 Node 测试。
- macOS 原有 launchd、Keychain、desktop refresh 行为无回归。

## Recommended First Execution Slice

第一轮实现只做 Milestone 1：

- 新增平台工具。
- 让 Windows 前台 `remodex up/run` 明确可用。
- 抽象 auth opener 和 wake lock。
- 不做 Windows 后台服务。
- 不做 Codex Desktop Windows refresh。
- 不引入 credential native dependency。

这样可以最快验证“Windows 能不能作为 Codex bridge 主机被 iPhone 控制”，同时风险最低。