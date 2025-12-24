# Overview and Architecture

## Introduction

Declarative Workflows provide a YAML-based approach to defining complex multi-agent orchestrations without writing code. The workflow definition is loaded from a YAML file and executed using the same runtime engine as code-based workflows.

## Key Difference: Declarative vs. Code-Based

The fundamental difference is simple:

```csharp
// Code-based workflow - defined in C#
var workflow = new WorkflowBuilder()
    .AddExecutor("step1", new MyExecutor())
    .AddEdge("step1", "step2")
    .Build();

// Declarative workflow - defined in YAML
Workflow workflow = DeclarativeWorkflowBuilder.Build("Marketing.yaml", options);
```

Both approaches use the same execution engine, but declarative workflows separate the orchestration logic from the implementation.

## Architecture Overview

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                     YAML Workflow Definition                 │
│                    (Marketing.yaml, etc.)                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              DeclarativeWorkflowBuilder                      │
│  - Parses YAML using Bot.ObjectModel                        │
│  - Creates PowerFx formula state                            │
│  - Builds executor graph                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                 Workflow Execution Engine                    │
│  - Event streaming                                          │
│  - Checkpoint management                                    │
│  - State management                                         │
│  - Executor orchestration                                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                   Runtime Components                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Executors  │  │Agent Provider│  │State Storage │     │
│  │  (Actions)   │  │  (Azure AI)  │  │(Variables)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. DeclarativeWorkflowBuilder

The builder is responsible for transforming YAML into executable workflows:

```csharp
public static class DeclarativeWorkflowBuilder
{
    public static Workflow Build<TInput>(
        string workflowFile,
        DeclarativeWorkflowOptions options,
        Func<TInput, ChatMessage>? inputTransform = null)
}
```

**Key Responsibilities:**
- Parse YAML using `AdaptiveDialog` from Bot.ObjectModel
- Initialize PowerFx formula engine for expression evaluation
- Create executor instances for each action
- Build the execution graph with proper edges
- Configure event streaming and checkpointing

#### 2. DeclarativeWorkflowOptions

Configuration container for workflow execution:

```csharp
public class DeclarativeWorkflowOptions
{
    // Agent provider for Azure AI agents
    public WorkflowAgentProvider AgentProvider { get; set; }
    
    // Configuration for PowerFx expressions
    public IDictionary<string, object> Configuration { get; set; }
    
    // Checkpoint storage for state persistence
    public ICheckpointManager? CheckpointManager { get; set; }
    
    // Additional runtime options
    // ...
}
```

#### 3. WorkflowAgentProvider

Abstract base class for integrating with AI agent services:

```csharp
public abstract class WorkflowAgentProvider
{
    // Create a new conversation
    public abstract Task<string> CreateConversationAsync(CancellationToken cancellationToken);
    
    // Invoke an agent
    public abstract IAsyncEnumerable<AgentRunResponseUpdate> InvokeAgentAsync(
        string agentId,
        string? agentVersion,
        string? conversationId,
        IEnumerable<ChatMessage>? messages,
        IDictionary<string, object?>? inputArguments,
        CancellationToken cancellationToken);
        
    // Manage conversation messages
    public abstract Task<ChatMessage> CreateMessageAsync(/*...*/);
    public abstract Task<ChatMessage> GetMessageAsync(/*...*/);
    public abstract IAsyncEnumerable<ChatMessage> GetMessagesAsync(/*...*/);
}
```

#### 4. Action Executors

Each YAML action type has a corresponding executor implementation:

- **InvokeAzureAgentExecutor** - Invokes Azure AI agents
- **SetVariableExecutor** - Sets variable values
- **ConditionGroupExecutor** - Evaluates conditional branches
- **ForeachExecutor** - Iterates over collections
- **QuestionExecutor** - Prompts for human input
- And many more...

## Execution Model

### Workflow Lifecycle

```
1. Build Phase
   ├─ Parse YAML
   ├─ Create PowerFx state
   ├─ Instantiate executors
   └─ Build execution graph

2. Initialization Phase
   ├─ Initialize variables
   ├─ Load checkpoint (if resuming)
   └─ Prepare event stream

3. Execution Phase
   ├─ Trigger workflow start event
   ├─ Execute trigger actions
   ├─ Process executor graph
   │  ├─ Invoke executor
   │  ├─ Emit events
   │  ├─ Update state
   │  └─ Follow edges to next executors
   ├─ Handle external inputs (if any)
   └─ Create checkpoints (if configured)

4. Completion Phase
   ├─ Emit workflow output event
   ├─ Finalize state
   └─ Clean up resources
```

### Event Flow

Events are the primary mechanism for observing and interacting with workflow execution:

```
WorkflowStartedEvent
    │
    ├─→ DeclarativeActionInvokedEvent (action: setVariable_1)
    ├─→ DeclarativeActionCompletedEvent (action: setVariable_1)
    │
    ├─→ DeclarativeActionInvokedEvent (action: question_agent)
    │   └─→ ExecutorInvokedEvent
    │       └─→ AgentRunResponseUpdate (streaming)
    │       └─→ ExecutorCompletedEvent
    ├─→ DeclarativeActionCompletedEvent (action: question_agent)
    │
    ├─→ ExternalInputRequest (if needed)
    │   └─→ ExternalInputResponse (provided by caller)
    │
    └─→ WorkflowOutputEvent
```

### State Management

Workflows maintain two primary variable scopes:

#### System Variables
- **System.ConversationId** - Current conversation identifier
- **System.LastMessage** - The most recent input message
- Additional system-provided values

#### Local Variables
User-defined variables with the `Local.` prefix:
- **Local.InputTask** - User's input task
- **Local.AgentResponse** - Response from an agent
- **Local.TurnCount** - Custom counter
- Any variable defined with SetVariable actions

## PowerFx Expression Evaluation

PowerFx expressions enable dynamic behavior in workflows:

```yaml
# Set a variable using an expression
- kind: SetVariable
  variable: Local.NextSpeaker
  value: =Search(Local.AvailableAgents, Local.ProgressLedger.next_speaker.answer, name)

# Conditional expression
- kind: ConditionGroup
  conditions:
    - condition: =Local.TurnCount > 4
      actions:
        # ...

# String interpolation
- kind: SendActivity
  activity: "Ticket created: #{Local.TicketId}"
```

### Formula State Lifecycle

```csharp
// 1. Create formula engine with configuration
WorkflowFormulaState state = new(options.CreateRecalcEngine());

// 2. Initialize with workflow definition
state.Initialize(workflowElement.WrapWithBot(), options.Configuration);

// 3. During execution, formulas are evaluated
// Variables are accessed: Local.VariableName
// Expressions are computed: =Local.Count + 1
// Functions are called: =UserMessage(Local.Text)
```

## Integration Points

### Azure AI Foundry Integration

Declarative workflows integrate with Azure AI Foundry through the agent provider:

```csharp
var options = new DeclarativeWorkflowOptions
{
    AgentProvider = new AzureAgentProvider(
        projectEndpoint: "https://your-project.openai.azure.com",
        credential: new AzureCliCredential(),
        deploymentName: "gpt-4"
    )
};
```

The provider handles:
- Agent creation and invocation
- Conversation management
- Message threading
- Token-based authentication

### Checkpoint Integration

Checkpoints enable workflow pause/resume:

```csharp
var options = new DeclarativeWorkflowOptions
{
    CheckpointManager = new InMemoryCheckpointManager()
    // Or: new FileSystemJsonCheckpointStore("./checkpoints")
};

// During execution, checkpoints are automatically created
// at configurable points

// Resume from checkpoint
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);
var resumedWorkflow = workflow.ResumeFromCheckpoint(checkpoint);
```

## Workflow Graph Structure

Internally, declarative workflows create a directed graph:

```
┌──────────────┐
│   START      │
│   (Trigger)  │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│SetVariable_1 │
└──────┬───────┘
       │
       ↓
┌──────────────┐      ┌──────────────┐
│InvokeAgent_A │──→───│InvokeAgent_B │
└──────┬───────┘      └──────┬───────┘
       │                     │
       └──────────┬──────────┘
                  ↓
           ┌──────────────┐
           │ConditionGroup│
           └──────┬───────┘
                  │
         ┌────────┴────────┐
         ↓                 ↓
   ┌──────────┐      ┌──────────┐
   │ Branch A │      │ Branch B │
   └──────────┘      └──────────┘
```

### Edge Types

1. **Sequential Edges** - Default flow from one action to the next
2. **Conditional Edges** - Based on condition evaluation
3. **Loop Edges** - GotoAction creates backward edges
4. **Completion Edges** - From actions to workflow end

## Performance Considerations

### Async Execution
All executors are async and support cancellation:

```csharp
public abstract Task<object> ExecuteAsync(
    IWorkflowContext context,
    CancellationToken cancellationToken);
```

### Streaming Support
Agent responses stream in real-time:

```csharp
await foreach (var update in agentProvider.InvokeAgentAsync(/*...*/)
{
    // Process streaming updates
    yield return update;
}
```

### State Isolation
Each workflow run maintains isolated state, allowing concurrent executions.

## Error Handling

Workflows emit error events for failures:

```csharp
// Executor failures
ExecutorFailedEvent
    ├─ ExecutorId
    ├─ Exception details
    └─ Context information

// Workflow errors
WorkflowErrorEvent
    ├─ Error message
    └─ Stack trace (if available)
```

Error handling strategies:
1. **Catch and continue** - Log error, proceed with workflow
2. **Retry with backoff** - Attempt operation multiple times
3. **Escalate to human** - Request external intervention
4. **Graceful degradation** - Fall back to alternative path

## Observability

Workflows support OpenTelemetry integration:

```csharp
var options = new DeclarativeWorkflowOptions
{
    // Telemetry automatically enabled when configured
    // Traces include:
    // - Workflow execution spans
    // - Executor invocation spans
    // - Agent call spans
    // - Custom event attributes
};
```

Events and metrics are automatically emitted for:
- Workflow start/completion
- Action execution timing
- Agent invocation duration
- Variable state changes
- Error occurrences

## Next Steps

- Learn about [YAML Schema Reference](./02-yaml-schema-reference.md)
- Explore [Action Types Reference](./03-action-types-reference.md)
- Understand [Event Types and Processing](./04-event-types-and-processing.md)
