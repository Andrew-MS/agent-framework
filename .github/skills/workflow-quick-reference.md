# Declarative Workflows Quick Reference

## Fast Lookups for Common Questions

### File Paths (all in `docs/declarative-workflows/`)
- **README** → `00-README.md`
- **Actions** → `03-action-types-reference.md`
- **Events** → `04-event-types-and-processing.md`
- **External Requests** → `08-human-in-the-loop.md` + `04-event-types-and-processing.md`
- **Expressions** → `09-expressions-and-powerfx.md`
- **Examples** → `11-examples-and-samples.md`

---

## YAML Syntax Quick Reference

### Basic Structure
```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: my_workflow
  actions:
    - kind: ActionType
      id: action_id
```

### Variable References
```yaml
# System variables
conversationId: =System.ConversationId
value: =System.LastMessage.Text

# Local variables
variable: Local.MyVar
value: =Local.Counter + 1

# String interpolation
activity: "Hello, {Local.Name}!"
```

### Expressions
```yaml
# Literal
value: 42

# Expression (with =)
value: =Local.Count + 1

# Condition
condition: =Local.Status = "Complete"

# Boolean
condition: =Local.A And Not(Local.B)
```

---

## Common Actions Quick Reference

### Invoke Agent
```yaml
- kind: InvokeAzureAgent
  id: invoke_agent
  agent:
    name: MyAgent
  output:
    autoSend: true
```

### Set Variable
```yaml
- kind: SetVariable
  id: set_var
  variable: Local.Counter
  value: 0
```

### Conditional
```yaml
- kind: ConditionGroup
  id: check
  conditions:
    - condition: =Local.Value > 10
      id: if_high
      actions:
        - kind: SendActivity
          id: msg
          activity: "High value"
```

### Loop
```yaml
- kind: Foreach
  id: loop
  itemsProperty: =Local.Items
  actions:
    - kind: SendActivity
      id: process
      activity: "{ThisItem.name}"
```

### Question (External Input)
```yaml
- kind: Question
  id: ask
  prompt: "What is your name?"
  property: Local.UserName
```

---

## External Request Handling Quick Reference

### Pattern
1. Workflow emits `ExternalInputRequest`
2. Checkpoint created automatically
3. Workflow pauses
4. Provide `ExternalInputResponse`
5. Workflow resumes

### Detecting Request
```csharp
if (evt is ExternalInputRequest request)
{
    var message = request.AgentResponse.Message;
    // Get user input
    var response = new ExternalInputResponse(userInput);
    // Resume workflow
}
```

### Function Call Request
```csharp
if (evt is ExternalInputRequest request)
{
    var functionCalls = request.AgentResponse.Message.Contents
        .OfType<FunctionCallContent>();
    
    foreach (var call in functionCalls)
    {
        var result = await ExecuteFunction(call.Name, call.Arguments);
        var response = new ExternalInputResponse(
            new ChatMessage(ChatRole.Tool, 
                new FunctionResultContent(call.CallId, result))
        );
    }
}
```

### External Loop
```yaml
- kind: InvokeAzureAgent
  id: agent
  agent:
    name: InteractiveAgent
  input:
    externalLoop:
      when: =Not(Local.IsComplete)
  output:
    responseObject: Local.Status
```

**Full Documentation:** `08-human-in-the-loop.md`

---

## Event Handling Quick Reference

### Common Events
```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    switch (evt)
    {
        case WorkflowStartedEvent:
            // Workflow started
            break;
        
        case DeclarativeActionInvokedEvent invoked:
            // Action starting: invoked.ActionId
            break;
        
        case WorkflowOutputEvent output:
            // Output produced
            break;
        
        case ExternalInputRequest request:
            // External input needed ★
            break;
        
        case WorkflowErrorEvent error:
            // Error occurred
            break;
    }
}
```

**Full Documentation:** `04-event-types-and-processing.md`

---

## PowerFx Expression Quick Reference

### Operators
```yaml
# Arithmetic
value: =Local.A + Local.B
value: =Local.A - Local.B
value: =Local.A * Local.B
value: =Local.A / Local.B

# Comparison
condition: =Local.A = Local.B
condition: =Local.A <> Local.B
condition: =Local.A > Local.B

# Logical
condition: =Local.A And Local.B
condition: =Local.A Or Local.B
condition: =Not(Local.A)
```

### Common Functions
```yaml
# String
value: =Upper(Local.Text)
value: =Lower(Local.Text)
value: =Len(Local.Text)
value: =Concat(Local.Array, ", ")

# Collection
value: =Search(Local.Items, "name", name)
value: =Filter(Local.Items, isActive)
value: =CountRows(Local.Items)

# Conditional
value: =If(Local.Score > 80, "Pass", "Fail")
condition: =IsBlank(Local.Value)

# Workflow-specific
value: =UserMessage(Local.Text)
value: =MessageText(Local.Response)
```

**Full Documentation:** `09-expressions-and-powerfx.md`

---

## Building and Running Quick Reference

### Build Workflow
```csharp
var options = new DeclarativeWorkflowOptions
{
    AgentProvider = agentProvider,
    CheckpointManager = checkpointManager
};

Workflow workflow = DeclarativeWorkflowBuilder.Build(
    "workflow.yaml",
    options
);
```

### Run Workflow
```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    // Process events
}
```

### Resume from Checkpoint
```csharp
var checkpoint = await checkpointManager.GetCheckpointAsync(runId, checkpointId);
var resumed = workflow.ResumeFromCheckpoint(checkpoint);
await foreach (var evt in resumed.ResumeAsync(response))
{
    // Continue processing
}
```

**Full Documentation:** `10-workflow-execution-and-runtime.md`

---

## State Management Quick Reference

### Variables
```yaml
# Initialize
- kind: SetVariable
  id: init
  variable: Local.Counter
  value: 0

# Update
- kind: SetVariable
  id: update
  variable: Local.Counter
  value: =Local.Counter + 1

# Reset
- kind: ResetVariable
  id: reset
  variable: Local.Counter
```

### Checkpoints
- Created automatically before external input
- Stored via ICheckpointManager
- Can resume from checkpoint
- Contains full workflow state

**Full Documentation:** `05-state-management-and-checkpointing.md`

---

## Common Patterns Quick Reference

### Retry Pattern
```yaml
- kind: SetVariable
  id: init_attempts
  variable: Local.Attempts
  value: 0

- kind: InvokeAzureAgent
  id: try_operation
  agent:
    name: Agent
  output:
    responseObject: Local.Result

- kind: ConditionGroup
  id: check_retry
  conditions:
    - condition: =Not(Local.Result.success) And Local.Attempts < 3
      id: should_retry
      actions:
        - kind: SetVariable
          id: increment
          variable: Local.Attempts
          value: =Local.Attempts + 1
        - kind: GotoAction
          id: retry
          actionId: try_operation
```

### Approval Pattern
```yaml
- kind: Question
  id: ask_approval
  prompt: "Approve? (yes/no)"
  property: Local.Approval

- kind: ConditionGroup
  id: check_approval
  conditions:
    - condition: =Lower(Local.Approval) = "yes"
      id: approved
      actions:
        # Execute approved action
```

**Full Documentation:** `07-control-flow-patterns.md`, `12-best-practices-and-patterns.md`

---

## Troubleshooting Quick Reference

### Question Not Being Asked?
- Check: Question action in workflow?
- Check: ExternalInputRequest event emitted?
- Check: Handling ExternalInputRequest in code?

### Agent Not Responding?
- Check: AgentProvider configured?
- Check: Agent name matches deployed agent?
- Check: Credentials valid?

### Variables Not Working?
- Check: Variable initialized?
- Check: Using correct scope (Local. or System.)?
- Check: Expression syntax (= prefix)?

### Checkpoint Not Restoring?
- Check: CheckpointManager configured?
- Check: RunId and CheckpointId valid?
- Check: Checkpoint file exists?

**Full Documentation:** `10-workflow-execution-and-runtime.md` (Troubleshooting section)

---

## Quick Navigation by Task

| Task | Primary File | Line Numbers |
|------|--------------|--------------|
| Write YAML | 02 | All |
| Use InvokeAzureAgent | 03 | 21-95 |
| Handle external requests | 08 | 39-135 ★ |
| Set variables | 03 | 322-380 |
| Implement conditionals | 03 | 469-539 |
| Process events | 04 | 485-550 |
| Create checkpoints | 05 | 174-285 |
| Setup agent provider | 06 | 30-100 |
| Write PowerFx | 09 | 30-250 |
| Build workflow | 10 | 10-75 |
| See examples | 11 | All |

★ = External request handling thoroughly documented

---

## Last Updated

2024-12-24
