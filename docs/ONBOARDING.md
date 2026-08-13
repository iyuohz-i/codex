# Codex 项目入门指南

> 基于知识图谱自动生成 | 6458 节点 · 1556 边 · 615 批次

## 项目概述

**Codex** 是 OpenAI 开发的本地编码代理（coding agent），提供 CLI 和 IDE 集成。项目采用 Rust + TypeScript 混合架构，支持安全沙箱执行、多模型推理、插件扩展等特性。

### 技术栈

| 层次 | 技术 |
|------|------|
| **核心引擎** | Rust (codex-rs/) |
| **协议层** | TypeScript (app-server-protocol) |
| **构建系统** | Bazel + Cargo |
| **包管理** | pnpm (JS), Cargo (Rust) |
| **CI/CD** | GitHub Actions |
| **开发环境** | DevContainer (Ubuntu 24.04) |

---

## 架构层次

### 1. 核心引擎层 (codex-rs/)

Rust workspace，包含所有核心功能：

| Crate | 职责 |
|-------|------|
| **codex-core** | 核心逻辑：会话管理、模型客户端、工具执行、安全检查 |
| **codex-execpolicy** | 执行策略定义与评估 |
| **codex-extension-api** | 扩展系统 API |
| **codex-shell-command** | Shell 命令检测与解析 |
| **codex-app-server** | App Server 守护进程 |
| **codex-mcp-server** | MCP (Model Context Protocol) 服务 |

### 2. 协议层 (app-server-protocol/)

TypeScript 类型定义，用于前后端通信：

- **JSON Schema** (`schema/json/v2/`)：RPC 方法的请求/响应结构
- **TypeScript 类型** (`schema/typescript/v2/`)：类型安全的接口定义
- 使用 `ts-rs` 从 Rust 自动生成

### 3. 工具与配置层

| 目录 | 用途 |
|------|------|
| `tools/` | 自定义 lint 规则、构建工具 |
| `bazel/` | Bazel 构建规则 |
| `.devcontainer/` | 开发容器配置 |
| `.github/workflows/` | CI/CD 流水线 |

---

## 核心概念

### 会话与线程模型

- **Thread**：独立的对话上下文，包含完整的消息历史
- **Turn**：单次用户-代理交互
- **Session**：线程的持久化存储

### 安全与沙箱

- **SafetyCheck**：命令执行前的安全检查（AutoApprove / AskUser / Reject）
- **SandboxPolicy**：定义允许的操作（read-only / workspace-write / danger-full-access）
- **ExecPolicy**：细粒度的执行策略控制

### 扩展系统

- **Plugin**：可安装的扩展包，包含技能、钩子、MCP 服务器
- **Skill**：特定任务的指令集（如 code-review、babysit-pr）
- **Hook**：事件驱动的自动化（PreToolUse / PostToolUse）

### 模型与推理

- **ModelClient**：统一的模型调用接口
- **ReasoningEffort**：推理强度控制（low / medium / high）
- **ServiceTier**：服务层级（默认 / 优先级 / 弹性）

---

## 关键文件地图

### 核心入口

| 文件 | 说明 |
|------|------|
| `codex-rs/core/src/lib.rs` | codex-core crate 根，导出 CodexThread、ThreadManager、ModelClient |
| `codex-rs/Cargo.toml` | Rust workspace 配置，定义所有 crate 成员 |
| `MODULE.bazel` | Bazel 模块根，声明外部依赖 |

### 协议定义

| 文件 | 说明 |
|------|------|
| `codex-rs/app-server-protocol/schema/json/v2/*.json` | JSON Schema 定义（~120 个） |
| `codex-rs/app-server-protocol/schema/typescript/v2/*.ts` | TypeScript 类型（~330 个） |

### 配置与工具

| 文件 | 说明 |
|------|------|
| `codex-rs/clippy.toml` | Clippy lint 配置 |
| `codex-rs/rustfmt.toml` | 代码格式化规则 |
| `tools/argument-comment-lint/` | 自定义 lint 规则 |

---

## 复杂度热点 ⚠️

以下文件复杂度较高，修改时需谨慎：

### 1. 构建配置

- **`MODULE.bazel`** (complex)
  - Bazel 依赖声明，跨平台工具链配置
  - 修改影响全局构建

- **`codex-rs/Cargo.toml`** (complex)
  - Workspace 依赖版本管理
  - 需注意依赖冲突

### 2. 协议 Schema

- **`ExternalAgentConfig*.json`** (complex)
  - 外部代理配置导入/检测
  - 包含大量迁移类型定义

- **`ItemCompletedNotification.json`** (complex)
  - 工具执行完成通知
  - 包含 Collab Agent、FileChange 等复杂类型

### 3. 插件系统

- **`PluginInstalledResponse.json`** (complex)
  - 插件安装响应
  - 包含完整的插件元数据结构

---

## 依赖关系

### 核心依赖链

```
codex-core
  ├── codex-execpolicy (执行策略)
  ├── codex-extension-api (扩展接口)
  └── codex-shell-command (命令解析)

app-server
  ├── codex-core
  └── app-server-protocol (TypeScript 类型)
```

### 构建依赖

```
MODULE.bazel
  ├── rules_rust (Rust 构建规则)
  ├── llvm-v8 (V8 引擎)
  └── zstd (压缩库)

Cargo.toml
  ├── codex-rs/* (workspace crates)
  └── 共享依赖版本
```

---

## 开发环境设置

### 前置条件

1. **Rust 工具链**（通过 rustup）
2. **Node.js 22+**
3. **Bazel**（通过 Bazelisk）
4. **just**（任务运行器）

### 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/openai/codex.git
cd codex

# 2. 安装依赖
pnpm install

# 3. 构建 Rust 组件
cd codex-rs
cargo build

# 4. 运行测试
just test

# 5. 启动开发环境（可选）
# 使用 DevContainer 或本地环境
```

### 常用命令

```bash
# Rust
cargo fmt                    # 格式化
cargo clippy -- -D warnings  # Lint 检查
cargo test                   # 运行测试

# Bazel
bazel build //...            # 构建所有
bazel test //...             # 测试所有

# 任务运行器
just test                    # 运行测试
just fmt                     # 格式化代码
just lint                    # Lint 检查
```

---

## 学习路径建议

### 第 1 周：理解核心

1. 阅读 `README.md` 了解项目目标
2. 探索 `codex-rs/core/src/lib.rs` 理解核心 API
3. 运行 `cargo doc --open` 查看文档

### 第 2 周：深入协议

1. 查看 `app-server-protocol/schema/typescript/v2/` 了解 RPC 接口
2. 阅读 `Thread.ts`、`Turn.ts` 理解会话模型
3. 尝试调用 MCP 工具

### 第 3 周：扩展开发

1. 阅读 `codex-rs/ext/extension-api/` 了解扩展接口
2. 查看现有插件实现
3. 尝试编写简单技能

### 持续学习

- 关注 `CHANGELOG.md` 了解最新变更
- 参与 Code Review 学习代码规范
- 阅读 `AGENTS.md` 了解 AI 代理配置

---

## 常见问题

### Q: 如何添加新的 RPC 方法？

1. 在 Rust 中定义请求/响应类型
2. 使用 `ts-rs` 生成 TypeScript 类型
3. 在 `app-server-protocol/src/` 实现处理逻辑
4. 添加 JSON Schema 验证

### Q: 如何测试沙箱功能？

1. 使用 `.devcontainer/Dockerfile.secure` 构建安全环境
2. 配置 `bubblewrap` 沙箱
3. 运行集成测试验证隔离性

### Q: 如何调试模型调用？

1. 启用 `RUST_LOG=debug` 查看详细日志
2. 检查 `codex-core/src/client/` 中的请求逻辑
3. 使用 `codex-rs/models-manager/` 管理模型配置

---

## 资源链接

- **项目文档**：`docs/` 目录
- **API 参考**：`cargo doc --open`
- **贡献指南**：`docs/contributing.md`
- **安全政策**：`SECURITY.md`

---

*本指南由知识图谱自动生成，基于 6458 个节点和 1556 条边分析。*
*最后更新：2026-08-10*
