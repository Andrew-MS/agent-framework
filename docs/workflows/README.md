# Workflows: end-to-end operations guide

Use this guide to run, scale, and recover Agent Framework workflows (both code-first and declarative). It assumes zero prior knowledge.

## Workflow shapes
- **Code workflows**: built with `WorkflowBuilder` and executors (e.g., `FunctionExecutor`, `ChatProtocolExecutor`).
- **Declarative workflows**: YAML parsed into a `Workflow` via `DeclarativeWorkflowBuilder` and executed with the same runtime.
- **Execution modes**: streaming (`StreamingRun`) vs single-step (`Run`). Both are driven by `AsyncRunHandle`.

## How to run
- **In-process execution**: use `InProcessExecutionEnvironment` for local or service-hosted runs.
  - Create: `var env = WorkflowHostingExtensions.CreateInProcessExecutionEnvironment(streaming: true|false, enableConcurrentRuns: bool);`
  - Stream without checkpointing: `env.OpenStreamAsync(workflow)` or `env.StreamAsync(workflow, input)`.
  - One-shot step to next halt: `env.RunAsync(workflow, input)`.
- **Chat-aware runs**: When `DescribeProtocolAsync` returns chat semantics, enqueue a `TurnToken(emitEvents: true)` after the first message to emit conversation events automatically (see `InProcessExecutionEnvironment.BeginRunHandlingChatProtocolAsync`).

## Events and observability
All workflow events derive from `WorkflowEvent` (`Data` carries payload).
- Core lifecycle: `WorkflowStartedEvent`, `WorkflowOutputEvent`, `WorkflowWarningEvent`, `WorkflowErrorEvent`.
- Declarative action events: `DeclarativeActionInvokedEvent` and `DeclarativeActionCompletedEvent` with `ActionId`, `ActionType`, `ParentActionId`, `PriorActionId`.
- Conversation events: `ConversationUpdateEvent` (`ConversationId`, `IsWorkflow`), `MessageActivityEvent` (`Message`).
- External input flow (human/tool): `ExternalInputRequest` (contains `AgentRunResponse`/`ChatMessage`), respond with `ExternalInputResponse(Messages)`; the executor persists responses and sets `System.LastMessage`.
- Subscribe via your `IWorkflowContext.AddEventAsync` implementation or stream `WorkflowEvent` objects from `StreamingRun`.

## External events and pauses
- **Request ports**: When an action cannot proceed (e.g., missing tool call results), a `RequestPort` emits `ExternalInputRequest`. Resume by sending `ExternalInputResponse` with the required `ChatMessage` payloads.
- **Human-in-the-loop**: Declarative `RequestExternalInput` pauses until a response arrives; state is updated and execution continues.
- **Turn orchestration**: For chat protocols, enqueue `TurnToken` to advance turns and surface events without extra traffic.

## State and checkpointing
- **Checkpoint manager**: `CheckpointManager.CreateInMemory()` for ephemeral, or `CheckpointManager.CreateJson(ICheckpointStore<JsonElement>, JsonSerializerOptions?)` for durable stores.
- **Custom stores**: Implement `ICheckpointStore<T>` (create, retrieve, index checkpoints). Use `CheckpointManager.CreateJson(customStore, customJsonOptions)` to plug in your store and type system.
- **Running with checkpoints**:
  - Start: `env.StreamAsync(workflow, checkpointManager)` or `env.RunAsync(workflow, input, checkpointManager)`.
  - Resume: `env.ResumeStreamAsync(workflow, checkpointInfo, checkpointManager)` or `env.ResumeAsync(...)` using a previously returned `CheckpointInfo`.
- **What is captured**: executor state, pending messages, and managed variable scopes. Declarative workflows also persist `Local`, `Global`, and `System` Power Fx scopes through `WorkflowFormulaState.RestoreAsync`.
- **Choosing durability**: use `InMemoryCheckpointManager` for tests; `FileSystemJsonCheckpointStore` for single-node durability; implement `ICheckpointStore<T>` (e.g., blob/DB) for distributed resiliency.

## Distributed and durable service pattern
1) **Build workflow** (code or YAML -> `Workflow`).
2) **Choose persistence**: `CheckpointManager` with your `ICheckpointStore` (backed by cloud/DB) and JSON marshaller options for custom types.
3) **Host runner**: create `InProcessExecutionEnvironment` (or your `IWorkflowExecutionEnvironment` implementation) in a stateless service.
4) **Start runs** with `StreamAsync`/`RunAsync`, store returned `CheckpointInfo`.
5) **Handle events**: subscribe to `WorkflowEvent` stream for telemetry, UI, or to forward `ExternalInputRequest` to humans/tools.
6) **Resume** from checkpoints on new nodes using `ResumeStreamAsync`/`ResumeAsync` with stored `CheckpointInfo`.
7) **External input**: when `ExternalInputRequest` arrives, collect tool/human data, send `ExternalInputResponse` to continue.

## Usage quick reference
- Minimal streaming run: `var run = await env.OpenStreamAsync(workflow);`
- Stream with durable checkpoints: `var run = await env.StreamAsync(workflow, checkpointManager); var checkpoint = run.Checkpoint.Info;`
- Resume: `var resumed = await env.ResumeStreamAsync(workflow, checkpoint, checkpointManager);`
- Declarative pause/resume: send `ExternalInputResponse` to the `RequestExternalInput` action; it writes `System.LastMessage` and assigned variables, then continues.

## File map
- Runtime & execution: `dotnet/src/Microsoft.Agents.AI.Workflows/**`
- Declarative runtime: `dotnet/src/Microsoft.Agents.AI.Workflows.Declarative/**`
- Checkpointing abstractions: `dotnet/src/Microsoft.Agents.AI.Workflows/Checkpointing/**`
- In-process hosting: `dotnet/src/Microsoft.Agents.AI.Workflows/InProc/**`
- Samples: `dotnet/samples/GettingStarted/Workflows` and `workflow-samples/*.yaml`
