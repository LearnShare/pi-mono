# pi-coding-agent Error Handling and Recovery Mechanisms

## Executive Summary

pi-coding-agent implements a layered error handling architecture spanning four core levels: provider-level error transformation, agent loop error propagation, agent-level state management, and session-level intelligent recovery. The system employs regex-based error classification to distinguish between retryable errors (rate limiting, server errors, network issues) and non-retryable errors (authentication failures, context overflow), providing appropriate recovery strategies for each category. Context overflow triggers automatic compaction with context summarization, while transient errors trigger exponential backoff retry (default: 3 retries, 2000ms base delay). Tool execution errors are captured and returned as tool result messages, allowing the agent to continue operation rather than terminate.

## 1. Architecture Overview

### 1.1 Error Handling Layers

The error handling system operates across four distinct layers, each with specific responsibilities:

| Layer | Location | Responsibility |
|-------|----------|----------------|
| Provider | packages/ai/src/providers/*.ts | Error transformation, status code mapping |
| Agent Loop | packages/agent/src/agent-loop.ts | Error propagation, tool execution handling |
| Agent | packages/agent/src/agent.ts | State management, failure message creation |
| Session | packages/coding-agent/src/core/agent-session.ts | Recovery strategies, retry/compaction logic |

### 1.2 Error Flow

```
LLM Provider
    |
    v (Provider error transformation)
Provider Error Message (stopReason="error", errorMessage="...")
    |
    v (Agent Loop propagates)
AgentMessage with stopReason and errorMessage
    |
    v (Session classifies and decides recovery)
    +-- Retryable Error --> Exponential Backoff Retry
    +-- Context Overflow --> Auto-Compaction + Retry
    +-- Auth Error --> User Notification
    +-- Non-retryable --> Error Message in Session
```

## 2. Provider Error Classification

### 2.1 Error Categories

Provider implementations transform diverse error formats into a unified error representation with human-readable messages. The error classification system uses string pattern matching in session-level recovery logic.

| Category | Patterns Matched | Examples | HTTP Codes |
|---------|-----------------|-----------|------------|
| Authentication | N/A (handled before call) | Invalid API key, expired token | 401, 403 |
| Rate Limit | /rate.?limit|too many requests|429/ | Too many requests, throttling | 429 |
| Server Error | /500|502|503|504|server.?error|internal.?error/ | Internal server error, service unavailable | 500-504 |
| Service Unavailable | /service.?unavailable/ | Service temporarily unavailable | 503 |
| Network Error | /network.?error|connection.?(error|refused|lost)/ | Connection reset, DNS failure | N/A |
| Timeout | /timeout|timed? ?out/ | Request timeout | 408, N/A |
| Context Overflow | See Section 5 | Prompt exceeds context window | 400, 413 |

**Source**: packages/coding-agent/src/core/agent-session.ts:2403-2415

### 2.2 Provider Error Transformation

Providers wrap raw errors in standardized formats with human-readable prefixes to enable downstream pattern matching:

**Amazon Bedrock** (packages/ai/src/providers/amazon-bedrock.ts:287-309):
```typescript
const BEDROCK_ERROR_PREFIXES: Record<string, string> = {
  InternalServerException: "Internal server error",
  ModelStreamErrorException: "Model stream error",
  ValidationException: "Validation error",
  ThrottlingException: "Throttling error",
  ServiceUnavailableException: "Service unavailable",
};
```

**Anthropic** (packages/ai/src/providers/anthropic.ts:388-390):
```typescript
if (sse.event === "error") {
  throw new Error(sse.data);
}
```

### 2.3 Context Overflow Detection

Context overflow errors are detected through regex pattern matching in **packages/ai/src/utils/overflow.ts:28-49**:

```typescript
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,              // Anthropic
  /request_too_large/i,             // Anthropic
  /input is too long for/i,          // Amazon Bedrock
  /exceeds the context window/i,    // OpenAI
  /input token count.*exceeds/i,    // Google Gemini
  /maximum prompt length is \d+/i,   // xAI Grok
  /reduce the length of the messages/i,  // Groq
  // ... additional patterns
];
```

Non-overflow patterns are excluded (lines 60-64):
```typescript
const NON_OVERFLOW_PATTERNS = [
  /^(Throttling error|Service unavailable):/i,  // Bedrock throttling (not overflow)
  /rate limit/i,
  /too many requests/i,
];
```

Detection reliability varies by provider (see overflow.ts:75-95 for full reliability table).

## 3. Agent Loop Error Handling

### 3.1 Core Error Handling in runLoop()

**Location**: packages/agent/src/agent-loop.ts:155-234

The main loop function handles errors at two points:

**Turn Completion** (lines 194-198):
```typescript
if (message.stopReason === "error" || message.stopReason === "aborted") {
  await emit({ type: "turn_end", message, toolResults: [] });
  await emit({ type: "agent_end", messages: newMessages });
  return;
}
```

When the assistant message stopReason indicates error, the loop terminates immediately without further tool execution.

### 3.2 Stream Assistant Response Error Handling

**Location**: packages/agent/src/agent-loop.ts:240-333

Error event handling (lines 307-320):
```typescript
case "done":
case "error": {
  const finalMessage = await response.result();
  // ... handle final message
  return finalMessage;
}
```

### 3.3 Tool Execution Error Handling

Tool execution uses try-catch to prevent individual tool failures from terminating the agent:

**Tool Preparation** (packages/agent/src/agent-loop.ts:517-567):
- Validates tool existence
- Validates argument schema via `validateToolArguments()`
- Applies optional `beforeToolCall` hook
- Returns immediate error result on validation failure

**Tool Execution** (packages/agent/src/agent-loop.ts:569-604):
```typescript
try {
  const result = await prepared.tool.execute(...);
  return { result, isError: false };
} catch (error) {
  return {
    result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
    isError: true,
  };
}
```

**Post-Execution Hook** (packages/agent/src/agent-loop.ts:606-649):
```typescript
if (config.afterToolCall) {
  try {
    const afterResult = await config.afterToolCall(...);
    // Allow hook to modify result
  } catch (error) {
    result = createErrorToolResult(error instanceof Error ? error.message : String(error));
    isError = true;
  }
}
```

This ensures tool errors are captured as tool result messages rather than unhandled exceptions.

## 4. Agent-Level Error Management

### 4.1 Run Failure Handling

**Location**: packages/agent/src/agent.ts:463-478

```typescript
private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
  const failureMessage = {
    role: "assistant",
    content: [{ type: "text", text: "" }],
    api: this._state.model.api,
    provider: this._state.model.provider,
    model: this._state.model.id,
    usage: EMPTY_USAGE,
    stopReason: aborted ? "aborted" : "error",
    errorMessage: error instanceof Error ? error.message : String(error),
    timestamp: Date.now(),
  } satisfies AgentMessage;
  this._state.messages.push(failureMessage);
  this._state.errorMessage = failureMessage.errorMessage;
  await this.processEvents({ type: "agent_end", messages: [failureMessage] });
}
```

Key points:
- Creates a failure assistant message with `stopReason` set to "aborted" or "error"
- Stores error message in both message content and `errorMessage` field
- Emits `agent_end` event for listener notification

### 4.2 Turn-Level Error Tracking

**Location**: packages/agent/src/agent.ts:524-528

```typescript
case "turn_end":
  if (event.message.role === "assistant" && event.message.errorMessage) {
    this._state.errorMessage = event.message.errorMessage;
  }
  break;
```

## 5. Session-Level Error Recovery

### 5.1 Retryable Error Detection

**Location**: packages/coding-agent/src/core/agent-session.ts:2403-2415

```typescript
private _isRetryableError(message: AssistantMessage): boolean {
  if (message.stopReason !== "error" || !message.errorMessage) return false;

  // Context overflow is handled by compaction, not retry
  const contextWindow = this.model?.contextWindow ?? 0;
  if (isContextOverflow(message, contextWindow)) return false;

  const err = message.errorMessage;
  return /overloaded|provider.?returned.?error|rate.?limit|
    too many requests|429|500|502|503|504|service.?unavailable|
    server.?error|internal.?error|network.?error|connection.?error|
    connection.?refused|connection.?lost|other side closed|
    fetch failed|upstream.?connect|reset before headers|
    socket hang up|ended without|http2 request did not get a response|
    timed? out|timeout|terminated|retry delay/i.test(err);
}
```

### 5.2 Retry Strategy: Exponential Backoff

**Location**: packages/coding-agent/src/core/agent-session.ts:2421-2495

```typescript
private async _handleRetryableError(message: AssistantMessage): Promise<boolean> {
  const settings = this.settingsManager.getRetrySettings();
  if (!settings.enabled) {
    this._resolveRetry();
    return false;
  }

  this._retryAttempt++;

  if (this._retryAttempt > settings.maxRetries) {
    // Max retries exceeded
    this._emit({
      type: "auto_retry_end",
      success: false,
      attempt: this._retryAttempt - 1,
      finalError: message.errorMessage,
    });
    this._retryAttempt = 0;
    this._resolveRetry();
    return false;
  }

  const delayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);

  this._emit({
    type: "auto_retry_start",
    attempt: this._retryAttempt,
    maxAttempts: settings.maxRetries,
    delayMs,
    errorMessage: message.errorMessage || "Unknown error",
  });

  // Remove error message from agent state (keep in session for history)
  const messages = this.agent.state.messages;
  if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
    this.agent.state.messages = messages.slice(0, -1);
  }

  // Wait with exponential backoff (abortable)
  this._retryAbortController = new AbortController();
  try {
    await sleep(delayMs, this._retryAbortController.signal);
  } catch {
    // Aborted during sleep
    this._retryAttempt = 0;
    this._emit({ type: "auto_retry_end", success: false, attempt, finalError: "Retry cancelled" });
    this._resolveRetry();
    return false;
  }

  // Retry via continue()
  setTimeout(() => {
    this.agent.continue().catch(() => {});
  }, 0);

  return true;
}
```

**Default Retry Settings** (packages/coding-agent/src/core/settings-manager.ts:721-727):
```typescript
getRetrySettings(): { enabled: boolean; maxRetries: number; baseDelayMs: number } {
  return {
    enabled: this.getRetryEnabled(),
    maxRetries: this.settings.retry?.maxRetries ?? 3,
    baseDelayMs: this.settings.retry?.baseDelayMs ?? 2000,
  };
}
```

### 5.3 Retry Timing

| Attempt | Delay Formula | Delay (ms) |
|---------|---------------|-----------|
| 1 | baseDelayMs | 2000 |
| 2 | baseDelayMs * 2 | 4000 |
| 3 | baseDelayMs * 4 | 8000 |

Maximum delay can be capped via `maxRetryDelayMs` (provider-level) or `Agent.maxRetryDelayMs`.

### 5.4 Non-Retryable Errors

Errors NOT retried:
- **Context overflow** (handled by compaction)
- **Authentication failures** (401, 403 - user re-authenticates)
- **Malformed requests** (will fail again)
- **Resource not found** (404)

## 6. Context Overflow Handling

### 6.1 Compaction Trigger Conditions

**Location**: packages/coding-agent/src/core/agent-session.ts:1755-1833

Two trigger conditions:

**Case 1: Overflow Error** (lines 1782-1804)
```typescript
if (sameModel && isContextOverflow(assistantMessage, contextWindow)) {
  if (this._overflowRecoveryAttempted) {
    this._emit({ type: "compaction_end", reason: "overflow", ...,
      errorMessage: "Context overflow recovery failed after one compact-and-retry attempt." });
    return;
  }
  this._overflowRecoveryAttempted = true;
  // Remove error message from context
  await this._runAutoCompaction("overflow", true);  // willRetry=true
  return;
}
```

**Case 2: Threshold** (lines 1807-1832)
```typescript
let contextTokens: number;
if (assistantMessage.stopReason === "error") {
  // Estimate from last successful response
  const estimate = estimateContextTokens(messages);
  contextTokens = estimate.tokens;
} else {
  contextTokens = calculateContextTokens(assistantMessage.usage);
}
if (shouldCompact(contextTokens, contextWindow, settings)) {
  await this._runAutoCompaction("threshold", false);  // willRetry=false
}
```

### 6.2 Auto-Compaction Process

**Location**: packages/coding-agent/src/core/agent-session.ts:1838-2010

1. Generate summary of earliest messages via LLM
2. Replace messages with summary entry
3. If `willRetry=true`, remove error message and call `agent.continue()`
4. If `willRetry=false` and queued messages exist, kick the loop

### 6.3 Overflow Recovery Limitations

- Only one overflow recovery attempt per message (prevents infinite recovery)
- Uses `contextWindow` from model metadata
- Cannot recover if no model available or no auth configured

## 7. Tool Execution Error Handling

### 7.1 Tool Hooks Architecture

**Location**: packages/coding-agent/src/core/agent-session.ts:373-423

The session installs hooks on the Agent for tool interception:

```typescript
this.agent.beforeToolCall = async ({ toolCall, args }) => {
  const runner = this._extensionRunner;
  if (!runner.hasHandlers("tool_call")) return undefined;

  try {
    return await runner.emitToolCall({ type: "tool_call", ... });
  } catch (err) {
    throw new Error(`Extension failed, blocking execution: ${String(err)}`);
  }
};

this.agent.afterToolCall = async ({ toolCall, args, result, isError }) => {
  const runner = this._extensionRunner;
  if (!runner.hasHandlers("tool_result")) return undefined;

  const hookResult = await runner.emitToolResult({ type: "tool_result", ... });
  return hookResult ? { ...hookResult, isError: hookResult.isError ?? isError } : undefined;
};
```

### 7.2 Error Result Format

Tool errors are returned as tool result messages:

**Location**: packages/agent/src/agent-loop.ts:651-656
```typescript
function createErrorToolResult(message: string): AgentToolResult<any> {
  return {
    content: [{ type: "text", text: message }],
    details: {},
  };
}
```

The agent continues operation with error tool results included in context.

## 8. Configurable Settings

### 8.1 User-Configurable Retry Settings

**Location**: packages/coding-agent/src/core/settings-manager.ts

Retry settings in settings file:
```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 60000,
      "maxRetries": 3,
      "maxRetryDelayMs": 60000
    }
  }
}
```

Compaction settings:
```json
{
  "compaction": {
    "enabled": true,
    "thresholdPercent": 85,
    "minTokens": 10000,
    "reserveTokens": 4000
  }
}
```

## 9. Source File Summary

| File | Key Functions | Lines |
|------|---------------|-------|
| packages/agent/src/agent-loop.ts | runLoop, streamAssistantResponse, executeToolCalls, prepareToolCall, executePreparedToolCall | 155-234, 240-333, 338-604 |
| packages/agent/src/agent.ts | handleRunFailure | 463-478 |
| packages/ai/src/utils/overflow.ts | isContextOverflow, OVERFLOW_PATTERNS | 112-138 |
| packages/ai/src/providers/amazon-bedrock.ts | formatBedrockError, BEDROCK_ERROR_PREFIXES | 287-309 |
| packages/coding-agent/src/core/agent-session.ts | _isRetryableError, _handleRetryableError, _checkCompaction | 2403-2415, 2421-2495, 1755-1833 |
| packages/coding-agent/src/core/settings-manager.ts | getRetrySettings, getProviderRetrySettings | 721-735 |

## 10. Best Practices

### 10.1 For Users

1. **Enable retry** for transient provider issues: `retry.enabled: true`
2. **Increase maxRetries** for unreliable providers: `retry.maxRetries: 5`
3. **Enable compaction** for long sessions: `compaction.enabled: true`
4. **Adjust threshold** based on model context size:
   - Large context (128k+): thresholdPercent: 85
   - Small context (32k): thresholdPercent: 70

### 10.2 For Developers

1. **Error messages should be human-readable**: Providers must wrap errors in descriptive messages for downstream pattern matching
2. **Use specific prefixes** (e.g., "Throttling error:", "Internal server error:") to enable accurate error classification
3. **Tool errors should return results, not throw**: Catch exceptions and return error tool results
4. **Distinguish overflow from throttling**: Use clear prefixes like "Throttling error: too many tokens" vs. "Context too large"

### 10.3 Error Handling Checklist

| Error Type | Detection | Recovery | Retry Appropriate |
|------------|-----------|----------|-------------------|
| Rate Limit (429) | Regex match | Backoff retry | Yes |
| Server Error (500-504) | Regex match | Backoff retry | Yes |
| Network Error | Regex match | Backoff retry | Yes |
| Context Overflow | Regex match + usage | Compaction + retry | No (different path) |
| Auth Failure (401/403) | Pre-call validation | User notification | No |
| Timeout | Regex match | Backoff retry | Yes |

---

*Generated: 2026-05-06*
*Analysis scope: packages/agent/src/, packages/ai/src/providers/, packages/coding-agent/src/core/*