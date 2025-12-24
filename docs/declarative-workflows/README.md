# Declarative workflows: operator guide

Use this primer when you need to reason about Agent Framework declarative workflows without prior context. It outlines the YAML surface area, lifecycle events, checkpointing/state rules, agent provider contracts, and the external request flow.

## Where to look
- Declarative workflow runtime: `dotnet/src/Microsoft.Agents.AI.Workflows.Declarative`
- Sample YAML: `workflow-samples/*.yaml` and `dotnet/samples/GettingStarted/Workflows/Declarative/*/*.yaml`
- Sample console runner: `dotnet/samples/GettingStarted/Workflows/Declarative`

## YAML building blocks
- Root: `kind: Workflow`
- `trigger`: entry point with `kind` (e.g., `OnConversationStart`), `id`, and `actions` (ordered list).
- `actions`: each action has `kind` and `id`; many accept Power Fx expressions with `=` (e.g., `value: =System.LastMessage.Text`).
- Common stateful actions: `SetVariable`, `SetTextVariable`, `SetMultipleVariables`, `ResetVariable`, `ClearAllVariables`, `ParseValue`, `EditTable`/`EditTableV2`.
- Control-flow: `ConditionGroup`/`ConditionItem`, `Foreach`, `GotoAction`, `BreakLoop`, `ContinueLoop`, `EndWorkflow`.
- Conversation/agent actions: `InvokeAzureAgent`, `CreateConversation`, `AddConversationMessage`, `CopyConversationMessages`, `RetrieveConversationMessage(s)`, `SendActivity`, `Question`, `RequestExternalInput`.
- Variables use scoped names: `Local.*` (per action scope), `Topic.*` (dialog topic), `Global.*`, `System.*` (conversation metadata such as `System.ConversationId`, `System.LastMessage`), and `Environment.*`. Component scopes alias to `Local` internally.

### Expression & data notes
- Power Fx drives evaluations; `WorkflowExpressionEngine` binds scopes into the `RecalcEngine`.
- String interpolation in activities/messages uses `{Scope.Variable}`; Power Fx expressions use `=` prefix.
- Function calls requested by an agent but not locally available surface as `RequestPort` events backed by `ExternalInputRequest`/`ExternalInputResponse` (see below).

## Execution lifecycle & emitted events
Events derive from `WorkflowEvent` (`Data` payload available on every event) and are raised through `IWorkflowContext.AddEventAsync`:
- `DeclarativeActionInvokedEvent`: emitted before each action executes. Payload: `ActionId`, `ActionType`, `ParentActionId`, `PriorActionId`.
- `DeclarativeActionCompletedEvent`: emitted after discrete actions finish (suppressed for long-running actions like `RequestExternalInput`). Payload mirrors the invoked event.
- `ConversationUpdateEvent`: broadcasts conversation changes. Payload: `ConversationId`, `IsWorkflow` (true when the workflow conversation is external).
- `MessageActivityEvent`: broadcasts message text when surfaced to listeners.
- External input events (human/tool loop):
  - `ExternalInputRequest`: sent when `RequestExternalInput` runs; payload is an `AgentRunResponse` containing the originating `ChatMessage` (or empty message when used purely as a gate).
  - `ExternalInputResponse`: send this back to the workflow with `Messages` (list of `ChatMessage`). The executor persists messages to the workflow conversation (via the active `WorkflowAgentProvider`) and writes the last message into the configured variable.

## Handling external requests (human/tool handoff)
`RequestExternalInput` inserts a pause point in the plan:
1) Executor raises `ExternalInputRequest` and **does not** emit a completion event until a response arrives.
2) Respond with `ExternalInputResponse` containing one or more `ChatMessage` objects (role + content). Text-only flows can wrap a user message (`new ChatMessage(ChatRole.User, "...")`).
3) The executor:
   - Writes each response message into the workflow conversation if one is tracked.
   - Sets `System.LastMessage` to the final message.
   - Assigns the configured `variable` path (e.g., `Local.MyInput`) to a Power Fx list representing the messages.

## State, scopes, and checkpointing
- Managed scopes persisted in checkpoints: `Local`, `Global`, and `System` (`WorkflowFormulaState.RestorableScopes`).
- State updates use `QueueStateUpdateAsync`/`QueueClearScopeAsync`; values are stored as `PortableValue` then rebound into Power Fx so expressions see the latest data.
- `WorkflowFormulaState.RestoreAsync` hydrates managed scopes from checkpoint storage on resume; component scopes are aliased to `Local`.
- System scope defaults (see `SystemScope`): `ConversationId`, `Conversation` record, `LastMessage`, `UserLanguage`, environment values, etc.

## Agent provider contract
Implement `WorkflowAgentProvider` to bridge workflow actions to your agent platform:
- Conversation APIs: `CreateConversationAsync`, `CreateMessageAsync`, `GetMessageAsync`, `GetMessagesAsync` (with paging).
- Agent invocation: `InvokeAgentAsync(agentId, agentVersion?, conversationId?, messages?, inputArguments?)` streaming `AgentRunResponseUpdate`.
- Tooling flags: `Functions` (extra tools the agent may call), `AllowConcurrentInvocation` (parallel tool calls), `AllowMultipleToolCalls` (per-response).
- Providers are used by actions like `InvokeAzureAgent`, `RequestExternalInput` (to persist inbound messages), and conversation management actions.

## Practical event processing tips
- Subscribe to `WorkflowEvent` stream to drive UI, logging, or human-in-the-loop prompts.
- Use `DeclarativeActionInvokedEvent`/`Completed` for timeline visualization; pair with `ConversationUpdateEvent` to correlate with conversation IDs.
- When resuming from checkpoints, ensure your `IWorkflowContext` implementation restores managed scopes before executing pending actions to avoid Power Fx evaluation gaps.

## Minimal YAML examples
Human approval gate:
```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: approvals
  actions:
    - kind: RequestExternalInput
      id: capture_reason
      variable: Local.ApprovalMessages
    - kind: SendActivity
      id: echo
      activity: "Latest reply: {Local.ApprovalMessages}"
```

Agent invocation with shared conversation:
```yaml
- kind: InvokeAzureAgent
  id: invoke_writer
  conversationId: =System.ConversationId
  agent:
    name: WriterAgent
```
