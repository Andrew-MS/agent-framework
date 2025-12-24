# State Management and Checkpointing

## Overview

State management and checkpointing are critical features that enable declarative workflows to maintain context, persist progress, and resume execution after interruptions. This document covers variable scopes, state persistence, checkpoint creation, and recovery strategies.

---

## Variable Scopes

Declarative workflows support two primary variable scopes:

### System Variables

System variables are automatically provided by the workflow runtime and are read-only.

**Available System Variables:**

| Variable | Type | Description |
|----------|------|-------------|
| `System.ConversationId` | string | Current conversation identifier |
| `System.LastMessage` | ChatMessage | Most recent input message |
| `System.LastMessage.Text` | string | Text content of last message |

**Example Usage:**

```yaml
# Use system conversation ID
- kind: InvokeAzureAgent
  id: invoke_agent
  conversationId: =System.ConversationId
  agent:
    name: MyAgent

# Access last message
- kind: SetVariable
  id: store_input
  variable: Local.UserInput
  value: =System.LastMessage.Text
```

---

### Local Variables

Local variables are user-defined and maintain state throughout the workflow execution.

**Declaration and Usage:**

```yaml
# Initialize variable
- kind: SetVariable
  id: init_counter
  variable: Local.Counter
  value: 0

# Update variable
- kind: SetVariable
  id: increment
  variable: Local.Counter
  value: =Local.Counter + 1

# Use in expressions
- kind: ConditionGroup
  id: check_count
  conditions:
    - condition: =Local.Counter >= 5
      id: threshold_reached
      actions:
        - kind: SendActivity
          id: notify
          activity: "Threshold reached: {Local.Counter}"
```

**Variable Types:**

Local variables can store various data types:

```yaml
# String
- kind: SetVariable
  id: set_name
  variable: Local.UserName
  value: "John Doe"

# Number
- kind: SetVariable
  id: set_count
  variable: Local.Count
  value: 42

# Boolean
- kind: SetVariable
  id: set_flag
  variable: Local.IsComplete
  value: true

# Array
- kind: SetVariable
  id: set_list
  variable: Local.Items
  value: |-
    =["Item1", "Item2", "Item3"]

# Object
- kind: SetVariable
  id: set_config
  variable: Local.Config
  value: |-
    ={
      setting1: "value1",
      setting2: 42,
      nested: {key: "value"}
    }

# Complex object from agent response
- kind: InvokeAzureAgent
  id: invoke_agent
  agent:
    name: DataAgent
  output:
    responseObject: Local.AgentData
```

---

### Variable Lifecycle

```
Workflow Start
    ↓
Initialize Variables (SetVariable actions)
    ↓
Execute Actions (read/write variables)
    ↓
[Checkpoint Created] - State snapshot saved
    ↓
Continue Execution
    ↓
[External Input] - State persisted while paused
    ↓
Resume Execution - State restored
    ↓
Workflow Complete - Variables discarded
```

---

## State Persistence

### In-Memory State

By default, workflow state is maintained in memory:

```csharp
// State exists only during workflow execution
var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
await foreach (var evt in workflow.RunAsync(input))
{
    // State is active
}
// State is discarded after completion
```

**Characteristics:**
- Fast access
- No persistence overhead
- Lost on process termination
- Not suitable for long-running workflows

---

### Persisted State

For durable workflows, state can be persisted using checkpoints:

```csharp
var options = new DeclarativeWorkflowOptions
{
    CheckpointManager = new FileSystemJsonCheckpointStore("./checkpoints")
    // Or: new InMemoryCheckpointManager()
};

var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
```

---

## Checkpointing

### What is a Checkpoint?

A checkpoint is a snapshot of the workflow's complete state at a specific point in time, including:

- All variable values
- Current execution position
- Conversation history
- Pending external inputs
- Execution metadata

### CheckpointInfo Structure

```csharp
public sealed class CheckpointInfo
{
    public string RunId { get; }        // Unique identifier for the run
    public string CheckpointId { get; }  // Unique identifier for the checkpoint
}
```

---

### When are Checkpoints Created?

Checkpoints are automatically created at key points:

1. **Before External Input Requests**
   - When a Question action executes
   - When waiting for tool execution approval
   - Before any human-in-the-loop interaction

2. **At Configured Intervals**
   - Based on checkpoint policy configuration
   - After significant state changes

3. **Explicitly Requested**
   - Via checkpoint API calls

**Example Flow:**

```
Action: SetVariable
    ↓
Action: InvokeAgent
    ↓
Action: Question (requires human input)
    ↓
[CHECKPOINT CREATED] ← State saved here
    ↓
Emit ExternalInputRequest
    ↓
[WORKFLOW PAUSED]
    ↓
Receive ExternalInputResponse
    ↓
Resume from checkpoint
```

---

### Checkpoint Storage

#### File System Storage

Store checkpoints as JSON files:

```csharp
var checkpointStore = new FileSystemJsonCheckpointStore("./checkpoints");

var options = new DeclarativeWorkflowOptions
{
    CheckpointManager = checkpointStore
};
```

**File Structure:**
```
checkpoints/
├── run-abc123/
│   ├── checkpoint-1.json
│   ├── checkpoint-2.json
│   └── checkpoint-3.json
└── run-def456/
    └── checkpoint-1.json
```

---

#### In-Memory Storage

Suitable for development and testing:

```csharp
var checkpointManager = new InMemoryCheckpointManager();

var options = new DeclarativeWorkflowOptions
{
    CheckpointManager = checkpointManager
};
```

**Characteristics:**
- Fast access
- No I/O overhead
- Lost on process restart
- Useful for testing

---

#### Custom Storage

Implement `ICheckpointManager` for custom storage:

```csharp
public interface ICheckpointManager
{
    Task<Checkpoint?> GetCheckpointAsync(
        string runId,
        string checkpointId,
        CancellationToken cancellationToken = default);
    
    Task SaveCheckpointAsync(
        Checkpoint checkpoint,
        CancellationToken cancellationToken = default);
    
    Task<IEnumerable<CheckpointInfo>> ListCheckpointsAsync(
        string runId,
        CancellationToken cancellationToken = default);
}
```

**Custom Implementations:**
- Azure Blob Storage
- Amazon S3
- Database (SQL, CosmosDB)
- Redis cache

---

### Creating Checkpoints

Checkpoints are primarily created automatically, but can be accessed programmatically:

```csharp
// Configure checkpoint policy
var options = new DeclarativeWorkflowOptions
{
    CheckpointManager = checkpointManager,
    // Additional checkpoint configuration
};

await foreach (var evt in workflow.RunAsync(input))
{
    // Checkpoint created automatically before external input
    if (evt is ExternalInputRequest)
    {
        // Checkpoint has been saved at this point
        var checkpoints = await checkpointManager.ListCheckpointsAsync(runId);
        var latest = checkpoints.Last();
        Console.WriteLine($"Checkpoint created: {latest.CheckpointId}");
    }
}
```

---

### Resuming from Checkpoint

Resume a workflow from a previously saved checkpoint:

```csharp
// Retrieve checkpoint
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);

if (checkpoint != null)
{
    // Resume workflow from checkpoint
    var resumedWorkflow = workflow.ResumeFromCheckpoint(checkpoint);
    
    // Provide external input that was requested
    var response = new ExternalInputResponse("User's response");
    
    await foreach (var evt in resumedWorkflow.ResumeAsync(response))
    {
        // Continue processing events
    }
}
```

---

### Checkpoint Contents

A checkpoint contains:

```json
{
  "runId": "abc123",
  "checkpointId": "xyz789",
  "timestamp": "2024-12-24T18:30:00Z",
  "state": {
    "variables": {
      "Local.Counter": 5,
      "Local.UserName": "John",
      "Local.IsComplete": false,
      "Local.AgentResponse": {
        "message": "...",
        "data": { ... }
      }
    },
    "executionPosition": {
      "currentActionId": "question_user",
      "completedActions": ["set_var1", "invoke_agent1", ...]
    },
    "conversationHistory": [
      {
        "role": "user",
        "content": "..."
      },
      {
        "role": "assistant",
        "content": "..."
      }
    ]
  },
  "pendingRequests": [
    {
      "type": "ExternalInputRequest",
      "data": { ... }
    }
  ]
}
```

---

## State Management Patterns

### Counter Pattern

Track iterations or attempts:

```yaml
# Initialize
- kind: SetVariable
  id: init_counter
  variable: Local.AttemptCount
  value: 0

# Increment
- kind: SetVariable
  id: increment
  variable: Local.AttemptCount
  value: =Local.AttemptCount + 1

# Check threshold
- kind: ConditionGroup
  id: check_limit
  conditions:
    - condition: =Local.AttemptCount > 3
      id: max_reached
      actions:
        - kind: SendActivity
          id: notify_max
          activity: "Maximum attempts reached"
        - kind: EndWorkflow
          id: stop
```

---

### Accumulator Pattern

Collect data across multiple steps:

```yaml
# Initialize empty array
- kind: SetVariable
  id: init_results
  variable: Local.Results
  value: =[]

# Add item
- kind: SetVariable
  id: add_result
  variable: Local.Results
  value: =Concat(Local.Results, [Local.CurrentResult])

# Process all results
- kind: SetVariable
  id: count_results
  variable: Local.TotalResults
  value: =CountRows(Local.Results)
```

---

### State Machine Pattern

Track workflow state:

```yaml
# Initialize state
- kind: SetVariable
  id: init_state
  variable: Local.State
  value: "Initializing"

# Transition states
- kind: ConditionGroup
  id: state_machine
  conditions:
    - condition: =Local.State = "Initializing"
      id: state_init
      actions:
        - kind: SetVariable
          id: to_processing
          variable: Local.State
          value: "Processing"
    
    - condition: =Local.State = "Processing"
      id: state_process
      actions:
        - kind: InvokeAzureAgent
          id: process_agent
          agent:
            name: ProcessorAgent
        
        - kind: SetVariable
          id: to_complete
          variable: Local.State
          value: "Complete"
```

---

### Caching Pattern

Store expensive computations:

```yaml
# Check cache
- kind: ConditionGroup
  id: check_cache
  conditions:
    - condition: =IsBlank(Local.CachedResult)
      id: cache_miss
      actions:
        # Compute result
        - kind: InvokeAzureAgent
          id: compute
          agent:
            name: ComputeAgent
          output:
            responseObject: Local.CachedResult

# Use cached result
- kind: SendActivity
  id: use_cached
  activity: "Result: {Local.CachedResult}"
```

---

## Error Recovery

### Checkpoint-Based Recovery

Use checkpoints to recover from failures:

```csharp
string runId = "workflow-run-123";

try
{
    await foreach (var evt in workflow.RunAsync(input))
    {
        // Process events
    }
}
catch (Exception ex)
{
    Console.WriteLine($"Workflow failed: {ex.Message}");
    
    // Get latest checkpoint
    var checkpoints = await checkpointManager.ListCheckpointsAsync(runId);
    var latest = checkpoints.LastOrDefault();
    
    if (latest != null)
    {
        Console.WriteLine($"Recovering from checkpoint: {latest.CheckpointId}");
        
        // Load checkpoint
        var checkpoint = await checkpointManager.GetCheckpointAsync(
            latest.RunId, 
            latest.CheckpointId
        );
        
        // Resume from checkpoint
        var resumedWorkflow = workflow.ResumeFromCheckpoint(checkpoint);
        
        // Continue execution
        await foreach (var evt in resumedWorkflow.ResumeAsync())
        {
            // Process events
        }
    }
}
```

---

### State Validation

Validate state integrity after resuming:

```csharp
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);

// Validate checkpoint before resuming
if (checkpoint != null && ValidateCheckpointState(checkpoint))
{
    var workflow = DeclarativeWorkflowBuilder.Build("workflow.yaml", options);
    var resumed = workflow.ResumeFromCheckpoint(checkpoint);
    // Continue...
}

bool ValidateCheckpointState(Checkpoint checkpoint)
{
    // Check required variables exist
    // Verify state consistency
    // Validate conversation history
    return true;
}
```

---

## State Cleanup

### Manual Cleanup

```csharp
// Delete old checkpoints
var allCheckpoints = await checkpointManager.ListCheckpointsAsync(runId);
var oldCheckpoints = allCheckpoints
    .Where(c => c.Timestamp < DateTime.UtcNow.AddDays(-30));

foreach (var old in oldCheckpoints)
{
    await checkpointManager.DeleteCheckpointAsync(old.RunId, old.CheckpointId);
}
```

### Reset Variables

```yaml
# Clear specific variable
- kind: ResetVariable
  id: clear_temp
  variable: Local.TempData

# Clear all variables
- kind: ClearAllVariables
  id: reset_all
```

---

## Performance Considerations

### Checkpoint Size

Minimize checkpoint size:

1. **Avoid Large Variables** - Don't store entire datasets in variables
2. **Use References** - Store IDs instead of full objects
3. **Clean Temp Data** - Reset temporary variables when done

### Checkpoint Frequency

Balance between recoverability and performance:

- **More Checkpoints** = Better recovery, higher overhead
- **Fewer Checkpoints** = Less overhead, more work to redo

### State Access Patterns

Optimize variable access:

```yaml
# ✅ Good - Calculate once, store result
- kind: SetVariable
  id: compute_summary
  variable: Local.Summary
  value: =ProcessData(Local.LargeDataset)

- kind: SendActivity
  id: use_summary
  activity: "{Local.Summary}"

# ❌ Avoid - Recalculating in every action
- kind: SendActivity
  id: recalc1
  activity: "{ProcessData(Local.LargeDataset)}"

- kind: SendActivity
  id: recalc2
  activity: "{ProcessData(Local.LargeDataset)}"
```

---

## Security Considerations

### Sensitive Data

Be cautious with sensitive data in checkpoints:

```yaml
# ❌ Avoid storing passwords directly
- kind: SetVariable
  id: bad_practice
  variable: Local.Password
  value: "secret123"

# ✅ Better - Store encrypted or use secure references
- kind: SetVariable
  id: better_practice
  variable: Local.TokenReference
  value: =GetSecureToken()
```

### Checkpoint Encryption

Implement encryption for checkpoint storage:

```csharp
public class EncryptedCheckpointStore : ICheckpointManager
{
    public async Task SaveCheckpointAsync(Checkpoint checkpoint, ...)
    {
        var json = JsonSerializer.Serialize(checkpoint);
        var encrypted = Encrypt(json);
        await _storage.WriteAsync(encrypted);
    }
    
    public async Task<Checkpoint?> GetCheckpointAsync(...)
    {
        var encrypted = await _storage.ReadAsync();
        var json = Decrypt(encrypted);
        return JsonSerializer.Deserialize<Checkpoint>(json);
    }
}
```

---

## Best Practices

1. **Name Variables Clearly** - Use descriptive names like `Local.UserAuthenticated`
2. **Initialize Variables** - Set initial values for all variables
3. **Validate State** - Check variable existence before use
4. **Clean Up Temporary State** - Reset variables when no longer needed
5. **Use Appropriate Storage** - Choose checkpoint storage based on durability needs
6. **Test Recovery** - Regularly test checkpoint resume scenarios
7. **Monitor Checkpoint Size** - Keep checkpoints reasonably sized
8. **Document State Schema** - Maintain documentation of variable structure
9. **Version Checkpoints** - Consider versioning if workflow schema changes
10. **Secure Sensitive Data** - Encrypt checkpoints containing sensitive information

---

## Troubleshooting

### Checkpoint Not Found

```csharp
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);
if (checkpoint == null)
{
    Console.WriteLine($"Checkpoint {checkpointId} not found for run {runId}");
    // Start workflow from beginning or use alternative checkpoint
}
```

### Invalid State After Resume

```csharp
try
{
    var resumed = workflow.ResumeFromCheckpoint(checkpoint);
    await foreach (var evt in resumed.ResumeAsync())
    {
        // Process events
    }
}
catch (InvalidStateException ex)
{
    Console.WriteLine($"Invalid state: {ex.Message}");
    // Checkpoint may be corrupted or incompatible with current workflow
}
```

### Variable Not Found

```yaml
# Check if variable exists before use
- kind: ConditionGroup
  id: check_var
  conditions:
    - condition: =Not(IsBlank(Local.OptionalVar))
      id: var_exists
      actions:
        - kind: SendActivity
          id: use_var
          activity: "{Local.OptionalVar}"
  
  elseActions:
    - kind: SendActivity
      id: no_var
      activity: "Variable not set"
```

---

## Next Steps

- Explore [Agent Providers and Integration](./06-agent-providers-and-integration.md)
- Learn about [Human-in-the-Loop and External Input](./08-human-in-the-loop.md)
- Understand [Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md)
