# Event Types and Event Processing

## Overview

Events are the primary mechanism for observing and monitoring workflow execution in declarative workflows. They provide real-time insights into workflow state, action execution, agent responses, and errors.

## Event Categories

1. **Workflow Events** - Workflow-level lifecycle events
2. **Declarative Action Events** - Action-specific events
3. **Executor Events** - Low-level executor execution events
4. **Agent Events** - Agent invocation and response events
5. **External Input Events** - Human-in-the-loop events
6. **Error and Warning Events** - Failure and diagnostic events

---

## Event Hierarchy

All events inherit from the base `WorkflowEvent` class:

```
WorkflowEvent (base)
├── ExecutorEvent
│   ├── ExecutorInvokedEvent
│   ├── ExecutorCompletedEvent
│   └── ExecutorFailedEvent
├── DeclarativeActionInvokedEvent
├── DeclarativeActionCompletedEvent
├── WorkflowStartedEvent
├── WorkflowOutputEvent
├── SuperStepEvent
│   └── SuperStepStartedEvent
├── AgentRunResponseEvent
├── AgentRunUpdateEvent
├── RequestInfoEvent
├── ExternalInputRequest
├── ExternalInputResponse
├── MessageActivityEvent
├── ConversationUpdateEvent
├── WorkflowErrorEvent
├── WorkflowWarningEvent
├── SubworkflowErrorEvent
└── SubworkflowWarningEvent
```

---

## Workflow Events

### WorkflowStartedEvent

Emitted when a workflow begins execution.

**Properties:**
- `Data` (object?) - The input message that triggered the workflow

**When Emitted:**
- At the very start of workflow execution
- Before any actions are executed

**Example:**
```csharp
WorkflowStartedEvent(Data: ChatMessage)
```

**Usage:**
```csharp
await foreach (var evt in workflow.RunAsync("Hello"))
{
    if (evt is WorkflowStartedEvent started)
    {
        Console.WriteLine($"Workflow started with: {started.Data}");
    }
}
```

---

### WorkflowOutputEvent

Emitted when an executor yields output to the user or external system.

**Properties:**
- `Data` (object) - The output data (typically ChatMessage)
- `SourceId` (string) - ID of the executor that produced the output

**Methods:**
- `Is<T>()` - Check if data is of type T
- `Is<T>(out T? value)` - Check and retrieve typed data
- `As<T>()` - Get data as type T (returns null if not compatible)

**When Emitted:**
- When an agent produces a response
- When SendActivity emits a message
- When any action produces user-facing output

**Example:**
```csharp
await foreach (var evt in workflow.RunAsync("input"))
{
    if (evt is WorkflowOutputEvent output)
    {
        if (output.Is<ChatMessage>(out var message))
        {
            Console.WriteLine($"Output from {output.SourceId}: {message.Text}");
        }
    }
}
```

---

### WorkflowErrorEvent

Emitted when a workflow-level error occurs.

**Properties:**
- `Data` (object?) - Error details or exception

**When Emitted:**
- When an unhandled exception occurs
- When a critical workflow failure happens

**Example:**
```csharp
if (evt is WorkflowErrorEvent error)
{
    Console.WriteLine($"Workflow error: {error.Data}");
}
```

---

### WorkflowWarningEvent

Emitted for non-critical issues during workflow execution.

**Properties:**
- `Data` (object?) - Warning details

**When Emitted:**
- For recoverable issues
- For deprecation warnings
- For configuration issues

---

## Declarative Action Events

### DeclarativeActionInvokedEvent

Emitted when a declarative action begins execution.

**Properties:**
- `ActionId` (string) - Unique identifier of the action
- `ActionType` (string) - Type name of the action (e.g., "InvokeAzureAgent")
- `ParentActionId` (string?) - ID of parent action if nested
- `PriorActionId` (string?) - ID of the previous action in sequence

**When Emitted:**
- Immediately before an action executes
- After the previous action has completed

**Example:**
```csharp
if (evt is DeclarativeActionInvokedEvent invoked)
{
    Console.WriteLine($"Executing {invoked.ActionType} (ID: {invoked.ActionId})");
    Console.WriteLine($"  Previous: {invoked.PriorActionId}");
    Console.WriteLine($"  Parent: {invoked.ParentActionId}");
}
```

**Event Flow:**
```
DeclarativeActionInvokedEvent(ActionId: "question_agent", ActionType: "InvokeAzureAgent")
```

---

### DeclarativeActionCompletedEvent

Emitted when a declarative action finishes execution.

**Properties:**
- `ActionId` (string) - Unique identifier of the action
- `ActionType` (string) - Type name of the action
- `ParentActionId` (string?) - ID of parent action if nested
- `PriorActionId` (string?) - ID of the previous action

**When Emitted:**
- After an action successfully completes
- Before the next action begins

**Example:**
```csharp
if (evt is DeclarativeActionCompletedEvent completed)
{
    Console.WriteLine($"Completed {completed.ActionType} (ID: {completed.ActionId})");
}
```

---

## Executor Events

### ExecutorInvokedEvent

Low-level event emitted when an executor handler is invoked.

**Properties:**
- `ExecutorId` (string) - Unique identifier of the executor
- `Data` (object) - The invocation message/input

**When Emitted:**
- Before executor logic executes
- For every executor in the workflow graph

**Example:**
```csharp
if (evt is ExecutorInvokedEvent invoked)
{
    Console.WriteLine($"Executor {invoked.ExecutorId} invoked");
}
```

---

### ExecutorCompletedEvent

Low-level event emitted when an executor completes.

**Properties:**
- `ExecutorId` (string) - Unique identifier of the executor
- `Data` (object?) - The result produced by the executor

**When Emitted:**
- After executor logic completes successfully
- Contains the executor's return value

**Example:**
```csharp
if (evt is ExecutorCompletedEvent completed)
{
    Console.WriteLine($"Executor {completed.ExecutorId} completed");
    if (completed.Data != null)
    {
        Console.WriteLine($"  Result: {completed.Data}");
    }
}
```

---

### ExecutorFailedEvent

Emitted when an executor encounters an error.

**Properties:**
- `ExecutorId` (string) - Unique identifier of the failed executor
- `Data` (object?) - Exception or error details

**When Emitted:**
- When an executor throws an exception
- When an executor returns an error result

**Example:**
```csharp
if (evt is ExecutorFailedEvent failed)
{
    Console.WriteLine($"Executor {failed.ExecutorId} failed");
    if (failed.Data is Exception ex)
    {
        Console.WriteLine($"  Error: {ex.Message}");
    }
}
```

---

## Agent Events

### AgentRunResponseEvent

Emitted when an agent produces a complete response.

**Properties:**
- `Data` (object?) - Agent response data (typically AgentRunResponse)

**When Emitted:**
- After an agent completes its execution
- Contains the full agent response with messages

**Example:**
```csharp
if (evt is AgentRunResponseEvent agentResponse)
{
    var response = agentResponse.Data as AgentRunResponse;
    Console.WriteLine($"Agent response: {response?.Message?.Text}");
}
```

---

### AgentRunUpdateEvent

Emitted during streaming agent responses.

**Properties:**
- `Data` (object?) - Partial update data

**When Emitted:**
- During agent streaming responses
- Multiple times for a single agent invocation
- Provides real-time updates as the agent generates its response

**Example:**
```csharp
if (evt is AgentRunUpdateEvent update)
{
    var updateData = update.Data as AgentRunResponseUpdate;
    // Process streaming update
}
```

---

## External Input Events

### ExternalInputRequest

Emitted when the workflow needs external input (human-in-the-loop).

**Properties:**
- `AgentResponse` (AgentRunResponse) - The message that triggered the request

**When Emitted:**
- When a Question action executes
- When an agent requests external tool execution
- When the workflow pauses for human approval

**Example:**
```csharp
if (evt is ExternalInputRequest request)
{
    Console.WriteLine($"External input requested");
    var message = request.AgentResponse.Message;
    
    // Prompt user for input
    var userInput = Console.ReadLine();
    
    // Create response
    var response = new ExternalInputResponse(userInput);
    // Resume workflow with response...
}
```

**Event Pattern:**
```
ExternalInputRequest
    ↓ (pause workflow)
  [User provides input]
    ↓
ExternalInputResponse
    ↓ (resume workflow)
```

---

### ExternalInputResponse

Provided by the caller to resume workflow after external input request.

**Properties:**
- `Messages` (IEnumerable<ChatMessage>) - Response messages

**When Used:**
- To provide user input to a Question action
- To provide tool execution results
- To approve/reject requested actions

**Example:**
```csharp
var response = new ExternalInputResponse(
    new ChatMessage(ChatRole.User, "My response")
);
```

---

## Communication Events

### MessageActivityEvent

Emitted when a SendActivity action sends a message.

**Properties:**
- Inherits from WorkflowEvent
- Contains message text in Data

**When Emitted:**
- When SendActivity action executes
- For workflow status updates
- For informational messages to the user

**Example:**
```csharp
if (evt is MessageActivityEvent activity)
{
    Console.WriteLine($"Activity: {activity.Data}");
}
```

---

### ConversationUpdateEvent

Emitted when conversation state changes.

**Properties:**
- Inherits from WorkflowEvent
- Contains conversation update details

**When Emitted:**
- When a conversation is created
- When messages are added to a conversation
- When conversation metadata changes

---

## Error and Warning Events

### SubworkflowErrorEvent

Emitted when an error occurs in a sub-workflow.

**Properties:**
- Inherits from WorkflowEvent
- Contains sub-workflow error details

**When Emitted:**
- When a nested workflow fails
- Provides context about the sub-workflow failure

---

### SubworkflowWarningEvent

Emitted for non-critical issues in sub-workflows.

**Properties:**
- Inherits from WorkflowEvent
- Contains warning details

**When Emitted:**
- For recoverable sub-workflow issues

---

## Event Processing Patterns

### Sequential Event Handling

Process events one at a time:

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case WorkflowStartedEvent started:
            Console.WriteLine("Workflow started");
            break;
        
        case DeclarativeActionInvokedEvent invoked:
            Console.WriteLine($"Action: {invoked.ActionId}");
            break;
        
        case WorkflowOutputEvent output:
            if (output.Is<ChatMessage>(out var message))
            {
                Console.WriteLine(message.Text);
            }
            break;
        
        case ExternalInputRequest inputRequest:
            // Handle input request
            break;
        
        case WorkflowErrorEvent error:
            Console.WriteLine($"Error: {error.Data}");
            break;
    }
}
```

---

### Filtering Specific Event Types

Use pattern matching and filtering:

```csharp
// Only process output events
await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is WorkflowOutputEvent output && output.Is<ChatMessage>(out var msg))
    {
        Console.WriteLine(msg.Text);
    }
}

// Only track action execution
await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        _actionTracker.TrackStart(invoked.ActionId, invoked.ActionType);
    }
    else if (evt is DeclarativeActionCompletedEvent completed)
    {
        _actionTracker.TrackEnd(completed.ActionId);
    }
}
```

---

### Event Aggregation

Collect events for batch processing:

```csharp
var events = new List<WorkflowEvent>();

await foreach (var evt in workflow.RunAsync(input))
{
    events.Add(evt);
    
    if (evt is WorkflowOutputEvent)
    {
        // Process accumulated events
        ProcessEventBatch(events);
        events.Clear();
    }
}
```

---

### Performance Monitoring

Track execution timing:

```csharp
var stopwatch = Stopwatch.StartNew();
var actionTimings = new Dictionary<string, long>();

await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case DeclarativeActionInvokedEvent invoked:
            actionTimings[invoked.ActionId] = stopwatch.ElapsedMilliseconds;
            break;
        
        case DeclarativeActionCompletedEvent completed:
            if (actionTimings.TryGetValue(completed.ActionId, out var startTime))
            {
                var duration = stopwatch.ElapsedMilliseconds - startTime;
                Console.WriteLine($"{completed.ActionId}: {duration}ms");
            }
            break;
    }
}
```

---

### Error Handling

Implement comprehensive error handling:

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    try
    {
        switch (evt)
        {
            case ExecutorFailedEvent failed:
                LogError($"Executor {failed.ExecutorId} failed", failed.Data);
                // Potentially retry or compensate
                break;
            
            case WorkflowErrorEvent workflowError:
                LogError("Workflow error", workflowError.Data);
                // Determine if workflow should continue
                break;
            
            case WorkflowWarningEvent warning:
                LogWarning("Workflow warning", warning.Data);
                // Log but continue
                break;
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing event: {ex.Message}");
    }
}
```

---

## Event Flow Example

Here's a typical event sequence for a simple workflow:

```
1. WorkflowStartedEvent
   └─ Data: "User's input message"

2. DeclarativeActionInvokedEvent
   └─ ActionId: "set_variable_1", ActionType: "SetVariable"

3. DeclarativeActionCompletedEvent
   └─ ActionId: "set_variable_1"

4. DeclarativeActionInvokedEvent
   └─ ActionId: "invoke_agent_1", ActionType: "InvokeAzureAgent"

5. ExecutorInvokedEvent
   └─ ExecutorId: "invoke_agent_1"

6. AgentRunUpdateEvent (multiple, streaming)
   └─ Data: Streaming response chunks

7. AgentRunResponseEvent
   └─ Data: Complete agent response

8. ExecutorCompletedEvent
   └─ ExecutorId: "invoke_agent_1"

9. WorkflowOutputEvent
   └─ Data: ChatMessage, SourceId: "invoke_agent_1"

10. DeclarativeActionCompletedEvent
    └─ ActionId: "invoke_agent_1"

11. WorkflowOutputEvent (final)
    └─ Data: Final workflow result
```

---

## Event Streaming

Workflows support async streaming of events:

```csharp
public async IAsyncEnumerable<WorkflowEvent> RunAsync(
    object input,
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    // Events are yielded as they occur
    // This allows real-time monitoring and interaction
}
```

**Benefits of Streaming:**
1. **Real-time Updates** - See progress as it happens
2. **Early Response** - Act on output before workflow completes
3. **Resource Efficiency** - Don't need to buffer all events
4. **Cancellation Support** - Can cancel mid-execution

---

## Event-Driven Patterns

### Progress Tracking

```csharp
int totalActions = 0;
int completedActions = 0;

await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is DeclarativeActionInvokedEvent)
        totalActions++;
    
    if (evt is DeclarativeActionCompletedEvent)
    {
        completedActions++;
        var progress = (completedActions * 100) / totalActions;
        Console.WriteLine($"Progress: {progress}%");
    }
}
```

### User Interface Updates

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case WorkflowOutputEvent output when output.Is<ChatMessage>(out var msg):
            await uiContext.DisplayMessageAsync(msg);
            break;
        
        case ExternalInputRequest request:
            var response = await uiContext.PromptUserAsync(request);
            // Resume workflow with response
            break;
    }
}
```

### Logging and Telemetry

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    logger.LogInformation($"Event: {evt.GetType().Name}");
    
    if (evt is DeclarativeActionInvokedEvent invoked)
    {
        telemetry.TrackEvent("ActionInvoked", new Dictionary<string, string>
        {
            ["ActionId"] = invoked.ActionId,
            ["ActionType"] = invoked.ActionType
        });
    }
}
```

---

## Best Practices

1. **Always Handle ExternalInputRequest** - Workflows will pause indefinitely without response
2. **Log Errors and Warnings** - Capture diagnostic events for troubleshooting
3. **Use Type Checking** - Leverage `Is<T>()` methods for safe type conversions
4. **Don't Block the Event Stream** - Process events quickly to avoid backpressure
5. **Consider Cancellation** - Support cancellation tokens for long-running workflows
6. **Buffer Carefully** - Streaming events shouldn't be buffered unnecessarily
7. **Track Action Lifecycle** - Match Invoked events with Completed/Failed events

---

## Next Steps

- Learn about [State Management and Checkpointing](./05-state-management-and-checkpointing.md)
- Understand [Human-in-the-Loop and External Input](./08-human-in-the-loop.md)
- Explore [Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md)
