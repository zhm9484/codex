# Context Compaction 策略详解

本文档详细剖析 Codex 仓库中的 **Context Compaction**（上下文压缩）机制，涵盖触发条件、执行流程、Prompt 设计、数据结构以及本地/远程两种压缩路径。

---

## 目录

1. [概述](#概述)
2. [核心文件索引](#核心文件索引)
3. [触发条件与时机](#触发条件与时机)
4. [阈值计算](#阈值计算)
5. [执行流程](#执行流程)
6. [本地压缩（Local Compaction）](#本地压缩local-compaction)
7. [远程压缩（Remote Compaction）](#远程压缩remote-compaction)
8. [Prompt 设计](#prompt-设计)
9. [压缩后的历史重建](#压缩后的历史重建)
10. [Initial Context 注入策略](#initial-context-注入策略)
11. [数据结构](#数据结构)
12. [容错与重试机制](#容错与重试机制)
13. [流程图](#流程图)

---

## 概述

Context Compaction 是 Codex 在对话过程中，当 token 使用量接近模型上下文窗口上限时，自动将历史对话压缩为摘要的机制。其核心思路是：

1. **监测** token 使用量是否达到阈值
2. **触发** 压缩任务（本地或远程）
3. **生成** 对话摘要（summary）
4. **重建** 精简的历史记录，替换原始冗长的对话历史
5. **注入** 新鲜的系统上下文（initial context），确保指令不过期

系统支持两种压缩路径：
- **本地压缩（Local）**：用当前模型自身生成摘要（适用于非 OpenAI 模型）
- **远程压缩（Remote）**：调用 OpenAI 的 `POST /responses/compact` 端点（适用于 OpenAI 模型）

---

## 核心文件索引

| 分类 | 文件路径 | 说明 |
|------|----------|------|
| 本地压缩核心 | `codex-rs/core/src/compact.rs` | 本地压缩逻辑、历史重建、用户消息收集 |
| 远程压缩核心 | `codex-rs/core/src/compact_remote.rs` | 远程压缩逻辑、历史过滤、API 调用 |
| 任务包装器 | `codex-rs/core/src/tasks/compact.rs` | `CompactTask` SessionTask 实现，路由到本地/远程 |
| 触发逻辑 | `codex-rs/core/src/codex.rs` | 自动/预采样/模型切换触发逻辑 |
| API 客户端 | `codex-rs/codex-api/src/endpoint/compact.rs` | `CompactClient`，调用 `responses/compact` 端点 |
| API 数据结构 | `codex-rs/codex-api/src/common.rs` | `CompactionInput` 定义 |
| 协议定义 | `codex-rs/protocol/src/protocol.rs` | `CompactedItem`、`TurnContextItem` |
| 协议 Item | `codex-rs/protocol/src/items.rs` | `ContextCompactionItem` |
| 模型配置 | `codex-rs/protocol/src/openai_models.rs` | `auto_compact_token_limit()` 计算 |
| 摘要 Prompt | `codex-rs/core/templates/compact/prompt.md` | 压缩摘要生成指令 |
| 摘要前缀 | `codex-rs/core/templates/compact/summary_prefix.md` | 注入压缩摘要时的前缀说明 |

---

## 触发条件与时机

Compaction 有 **四种触发时机**：

### 1. Pre-Turn 压缩（采样前）

**位置**：`codex.rs` → `run_pre_sampling_compact()`

在每次 turn 的模型请求发起**之前**检查。如果当前 token 使用量已超过 `auto_compact_token_limit`，则先压缩再采样。

```
run_turn() → run_pre_sampling_compact()
    ├── maybe_run_previous_model_inline_compact()  // 先检查模型切换
    └── if total_tokens >= auto_compact_limit → run_auto_compact(DoNotInject)
```

关键特征：
- 使用 `InitialContextInjection::DoNotInject`（不在压缩历史中注入 initial context）
- 下一次常规 turn 时会自动注入 initial context

### 2. Mid-Turn 压缩（采样中）

**位置**：`codex.rs` 的 sampling 循环中

在模型采样完成后检查。需要**同时满足两个条件**：
- `token_limit_reached`：总 token 使用量 ≥ `auto_compact_limit`
- `needs_follow_up`：当前 turn 还有后续工作（非 turn 结束点）

```rust
if token_limit_reached && needs_follow_up {
    run_auto_compact(sess, turn_context, InitialContextInjection::BeforeLastUserMessage)
}
```

关键特征：
- 使用 `InitialContextInjection::BeforeLastUserMessage`（在最后一条真实用户消息前注入 initial context）
- 模型训练数据期望 compaction summary 作为历史中的最后一个 item

### 3. 模型切换压缩

**位置**：`codex.rs` → `maybe_run_previous_model_inline_compact()`

当切换到**上下文窗口更小**的模型时触发。需同时满足：
- 模型 slug 发生变化
- 旧模型的 context_window > 新模型的 context_window
- 当前 total_tokens > 新模型的 `auto_compact_token_limit`

特殊之处：使用**前一个模型**的 `TurnContext` 执行压缩，因为前一个模型可能更适合处理当前（更大的）上下文。

### 4. 手动压缩

通过 API 显式触发（`ThreadCompactStartParams`），走 `CompactTask` → `run_compact_task()` / `run_remote_compact_task()` 路径。

---

## 阈值计算

**位置**：`openai_models.rs` → `ModelInfo::auto_compact_token_limit()`

```rust
pub fn auto_compact_token_limit(&self) -> Option<i64> {
    let context_limit = self
        .context_window
        .map(|context_window| (context_window * 9) / 10);  // 上下文窗口的 90%
    let config_limit = self.auto_compact_token_limit;
    if let Some(context_limit) = context_limit {
        return Some(
            config_limit.map_or(context_limit, |limit| std::cmp::min(limit, context_limit)),
        );
    }
    config_limit
}
```

计算规则：
1. **默认阈值**：模型 `context_window` 的 **90%**
2. **可配置覆盖**：通过 `auto_compact_token_limit` 字段在模型元数据中配置
3. **取较小值**：如果两者都存在，取 `min(configured_limit, 90% * context_window)`
4. **上下文窗口**还会乘以 `effective_context_window_percent`（通常 100%）

示例：如果模型 context_window = 200,000 tokens，则自动压缩阈值 = 180,000 tokens。

---

## 执行流程

### 路由决策

```rust
// compact.rs
pub(crate) fn should_use_remote_compact_task(provider: &ModelProviderInfo) -> bool {
    provider.is_openai()
}
```

- **OpenAI 模型** → 远程压缩（调用 `/responses/compact`）
- **其他模型** → 本地压缩（用模型自身生成摘要）

### 任务分发（tasks/compact.rs）

```rust
impl SessionTask for CompactTask {
    async fn run(...) {
        if should_use_remote_compact_task(&ctx.provider) {
            run_remote_compact_task(session, ctx).await
        } else {
            run_compact_task(session, ctx, input).await
        }
    }
}
```

---

## 本地压缩（Local Compaction）

**入口**：`compact.rs` → `run_inline_auto_compact_task()` / `run_compact_task()`

### 流程步骤

```
1. 创建 ContextCompactionItem（标记压缩事件开始）
2. 克隆当前对话历史
3. 将压缩 prompt 添加到历史末尾
4. 构建 Prompt（含 base_instructions + personality）
5. 调用模型流式生成摘要
   └── 循环重试：
       ├── 成功 → 跳出循环
       ├── ContextWindowExceeded → 移除最早的历史 item，重试
       └── 其他错误 → 指数退避重试（最多 max_retries 次）
6. 从模型响应中提取最后一条 assistant 消息作为摘要
7. 格式化摘要：SUMMARY_PREFIX + "\n" + 模型生成的摘要
8. 收集所有真实用户消息（过滤掉 summary、指令、环境上下文）
9. 构建替换历史（build_compacted_history）
10. 按需注入 initial context
11. 保留 GhostSnapshot（用于 /undo 功能）
12. 替换整个对话历史
13. 重新计算 token 使用量
14. 发出压缩完成事件 + 警告提示
```

### 摘要提取

```rust
let summary_suffix = get_last_assistant_message_from_turn(history_items)
    .unwrap_or_default();
let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

模型生成的摘要被嵌入到 `SUMMARY_PREFIX` 之后，以用户消息的形式存入替换历史中。

---

## 远程压缩（Remote Compaction）

**入口**：`compact_remote.rs` → `run_inline_remote_auto_compact_task()` / `run_remote_compact_task()`

### 流程步骤

```
1. 创建 ContextCompactionItem
2. 克隆当前对话历史
3. 预处理：裁剪 function call 历史以适应上下文窗口
   └── trim_function_call_history_to_fit_context_window()
       从末尾移除 codex 生成的 item 直到估算 token 数 ≤ context_window
4. 保留 GhostSnapshot
5. 构建 Prompt（包含完整历史 + base_instructions）
6. 调用 model_client.compact_conversation_history()
   └── 发送 POST /responses/compact
       请求体：CompactionInput { model, input, instructions }
       响应体：{ output: Vec<ResponseItem> }
7. 后处理：process_compacted_history()
   ├── 过滤（retain）：移除不需要的 item
   └── 注入 initial context（如果是 mid-turn）
8. 附加 GhostSnapshot
9. 替换对话历史
10. 重新计算 token 使用量
```

### 历史过滤规则（`should_keep_compacted_history_item`）

| Item 类型 | 保留？ | 原因 |
|-----------|--------|------|
| `developer` 消息 | 否 | 可能包含过期的指令/权限 |
| `user` 消息（真实用户输入） | 是 | 保留用户原始意图 |
| `user` 消息（系统生成：AGENTS.md、环境上下文、turn_aborted） | 否 | 属于 session prefix，会被重新注入 |
| `assistant` 消息 | 是 | 远程压缩模型可能生成 |
| `Compaction` item | 是 | 保留压缩链 |
| 函数调用、工具调用、推理 item 等 | 否 | 已被压缩为摘要 |
| `GhostSnapshot` | 否（但在外层单独保留） | 用于 /undo，在过滤后重新附加 |

---

## Prompt 设计

### 摘要生成 Prompt（Summarization Prompt）

**文件**：`codex-rs/core/templates/compact/prompt.md`

```
You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary
for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue

Be concise, structured, and focused on helping the next LLM seamlessly
continue the work.
```

**设计要点**：
- 将压缩框架为"交接给另一个 LLM"，鼓励模型生成自包含、结构化的摘要
- 明确要求包含四个维度：进度、上下文、待办事项、关键数据
- 强调简洁性，避免冗余

### 摘要前缀（Summary Prefix）

**文件**：`codex-rs/core/templates/compact/summary_prefix.md`

```
Another language model started to solve this problem and produced a summary
of its thinking process. You also have access to the state of the tools that
were used by that language model. Use this to build on the work that has
already been done and avoid duplicating work. Here is the summary produced
by the other language model, use the information in this summary to assist
with your own analysis:
```

**设计要点**：
- 告知"接替"的模型：有一个前任模型已经做了一部分工作
- 明确指出工具状态（文件修改等）仍然可用
- 要求在已有工作基础上继续，避免重复劳动

### Prompt 使用位置

1. **Summarization Prompt**：作为本地压缩的用户输入，传给模型要求其生成摘要
2. **Summary Prefix**：压缩完成后，拼接在模型生成的摘要文本前面，作为新历史中的用户消息

---

## 压缩后的历史重建

### `build_compacted_history()`

```rust
fn build_compacted_history(
    initial_context: Vec<ResponseItem>,  // 初始上下文（通常为空）
    user_messages: &[String],             // 收集的真实用户消息
    summary_text: &str,                   // SUMMARY_PREFIX + 模型摘要
) -> Vec<ResponseItem>
```

重建逻辑：

1. **Initial context**（如果有）放在最前面
2. **用户消息**：从最新到最旧遍历，累计不超过 `COMPACT_USER_MESSAGE_MAX_TOKENS`（20,000 tokens）
   - 如果单条消息超出剩余额度，进行截断
   - 反转后按时间顺序排列
3. **摘要文本**：作为最后一条 user 消息追加

### 用户消息收集（`collect_user_messages()`）

过滤规则：
- 只保留 `TurnItem::UserMessage` 类型的消息
- 排除以 `SUMMARY_PREFIX` 开头的消息（已有的压缩摘要）
- 排除系统生成的 session prefix（AGENTS.md 指令、环境上下文等）

---

## Initial Context 注入策略

**`InitialContextInjection` 枚举**控制压缩后是否以及如何注入 initial context（系统指令、权限、环境上下文等）：

### `DoNotInject`

- 用于：Pre-Turn 压缩、手动压缩
- 行为：不在替换历史中注入 initial context
- 原因：下一个常规 turn 开始时会自动注入最新的 initial context
- 同时清除 `reference_context_item`，确保完整重新注入

### `BeforeLastUserMessage`

- 用于：Mid-Turn 压缩
- 行为：在替换历史中，将 initial context 插入到**最后一条真实用户消息之前**
- 原因：mid-turn 压缩后模型会立即继续工作，需要看到最新的系统指令

### 插入位置优先级（`insert_initial_context_before_last_real_user_or_summary()`）

```
1. 最后一条真实用户消息之前（优先）
2. 最后一条摘要消息之前（如果没有真实用户消息）
3. 最后一条 Compaction item 之前（如果没有用户消息或摘要）
4. 追加到末尾（如果历史为空）
```

**压缩后历史结构（mid-turn 示例）**：

```
[保留的用户消息] → [Initial Context] → [最新用户消息] → [摘要]
```

---

## 数据结构

### `CompactedItem`

```rust
pub struct CompactedItem {
    pub message: String,                               // 摘要文本
    pub replacement_history: Option<Vec<ResponseItem>>, // 替换后的新历史
}
```

- 持久化到 rollout（对话转录记录）中
- `message`：本地压缩时为 `SUMMARY_PREFIX + 摘要`；远程压缩时为空字符串

### `ContextCompactionItem`

```rust
pub struct ContextCompactionItem {
    pub id: String,  // 唯一 ID，标识一次压缩事件
}
```

- 用于发出 `TurnItemStarted` / `TurnItemCompleted` 事件
- 让前端/客户端知道正在进行压缩

### `TurnContextItem`

- 记录当前 turn 的上下文快照
- 用于 resume/fork 场景下的回放（replay）
- mid-turn 压缩时保存，pre-turn 压缩时设为 `None`

### `CompactionInput`（API 请求体）

```rust
pub struct CompactionInput<'a> {
    pub model: &'a str,          // 模型标识
    pub input: &'a [ResponseItem], // 完整对话历史
    pub instructions: &'a str,     // 系统指令
}
```

---

## 容错与重试机制

### 本地压缩

| 错误类型 | 处理方式 |
|----------|----------|
| `ContextWindowExceeded` | 移除历史中最早的 item，重试（循环直到适配） |
| `Interrupted` | 立即返回错误，不重试 |
| 其他流式错误 | 指数退避重试，最多 `stream_max_retries` 次 |

裁剪后会发送通知：
> "Trimmed {N} older thread item(s) before compacting so the prompt fits the model context window."

### 远程压缩

- 发送前先裁剪（`trim_function_call_history_to_fit_context_window`）：
  - 从末尾移除 codex 生成的 item（函数调用输出等）
  - 直到估算 token 数 ≤ context_window
- 失败时记录详细日志：包括 token breakdown、context window 大小、请求字节数
- 错误通过事件传播到前端

### 压缩后警告

每次压缩成功后，系统都会发出警告：
> "Heads up: Long threads and multiple compactions can cause the model to be less accurate. Start a new thread when possible to keep threads small and targeted."

---

## 流程图

### 整体触发与路由

```
                    ┌─────────────┐
                    │  run_turn() │
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │ run_pre_sampling_compact │ ← 采样前检查
              └────────────┬────────────┘
                           │
          ┌────────────────▼────────────────┐
          │ maybe_run_previous_model_        │
          │        inline_compact()          │ ← 模型切换检查
          └────────────────┬────────────────┘
                           │
              ┌────────────▼────────────┐
              │ total_tokens ≥ limit?   │
              │ (pre-turn check)        │
              └───┬────────────────┬────┘
                  │ Yes            │ No
    ┌─────────────▼──────┐        │
    │ run_auto_compact() │        │
    │ (DoNotInject)      │        │
    └─────────────┬──────┘        │
                  │               │
              ┌───▼───────────────▼───┐
              │    模型采样 (Sampling)  │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ token_limit_reached   │
              │    && needs_follow_up?│
              └───┬───────────────┬───┘
                  │ Yes           │ No
    ┌─────────────▼──────┐       │
    │ run_auto_compact() │       │
    │ (BeforeLastUser)   │       │
    └─────────────┬──────┘       │
                  │              │
              ┌───▼──────────────▼───┐
              │    继续/结束 Turn     │
              └──────────────────────┘
```

### 压缩路由

```
         run_auto_compact()
                │
    ┌───────────▼───────────┐
    │ is_openai() provider? │
    └───┬───────────────┬───┘
        │ Yes           │ No
  ┌─────▼─────┐   ┌────▼────┐
  │  Remote    │   │  Local  │
  │ Compaction │   │Compaction│
  └─────┬─────┘   └────┬────┘
        │               │
  ┌─────▼─────┐   ┌────▼─────────────┐
  │ POST       │   │ Stream prompt to │
  │ /responses │   │ model, drain     │
  │ /compact   │   │ response         │
  └─────┬─────┘   └────┬─────────────┘
        │               │
  ┌─────▼───────────────▼─────┐
  │ process/build compacted   │
  │ history + inject context  │
  └─────────────┬─────────────┘
                │
  ┌─────────────▼─────────────┐
  │ replace_compacted_history │
  │ recompute_token_usage     │
  └───────────────────────────┘
```
