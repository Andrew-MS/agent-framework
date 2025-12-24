# Human-in-the-Loop and External Input

## Overview

Human-in-the-loop (HITL) capabilities allow workflows to pause execution and request input from users or external systems. This enables approval workflows, clarification requests, and interactive decision-making.

## Question Action

The simplest way to request human input is the `Question` action.

### Basic Question

```yaml
- kind: Question
  id: ask_name
  prompt: "What is your name?"
  property: Local.UserName

- kind: SendActivity
  id: greet
  activity: "Hello, {Local.UserName}!"
```

### Conditional Question

```yaml
- kind: ConditionGroup
  id: check_info
  conditions:
    - condition: =IsBlank(Local.UserEmail)
      id: need_email
      actions:
        - kind: Question
          id: ask_email
          prompt: "Please provide your email address:"
          property: Local.UserEmail
```

## External Input Pattern

When a workflow needs external input, it follows this pattern:

```
1. Workflow encounters Question or external loop
   ↓
2. Checkpoint created (workflow state saved)
   ↓
3. ExternalInputRequest event emitted
   ↓
4. Workflow pauses, waiting for response
   ↓
5. External system/user provides input
   ↓
6. ExternalInputResponse provided to workflow
   ↓
7. Workflow resumes from checkpoint
   ↓
8. Continue execution with user input
```

## External Loop

Agents can request repeated user input using external loops:

```yaml
- kind: InvokeAzureAgent
  id: interactive_agent
  agent:
    name: InteractiveAgent
  input:
    externalLoop:
      when: =Not(Local.IsComplete)
  output:
    autoSend: true
    responseObject: Local.Status
```

**How it works:**
- Agent executes and updates `Local.Status`
- Condition `Not(Local.IsComplete)` is evaluated
- If true, workflow emits `ExternalInputRequest`
- After user responds, agent invokes again
- Loop continues until condition becomes false

## Processing External Input Events

### Detecting Input Requests

```csharp
await foreach (var evt in workflow.RunAsync(input))
{
    if (evt is ExternalInputRequest request)
    {
        Console.WriteLine("Input requested:");
        var message = request.AgentResponse.Message;
        Console.WriteLine(message.Text);
        
        // Prompt user
        var userInput = Console.ReadLine();
        
        // Create response
        var response = new ExternalInputResponse(userInput);
        
        // Resume workflow (implementation depends on API)
    }
}
```

### Handling Function Call Requests

When an agent requests an unavailable tool:

```csharp
if (evt is ExternalInputRequest request)
{
    var functionCalls = request.AgentResponse.Message.Contents
        .OfType<FunctionCallContent>();
    
    foreach (var call in functionCalls)
    {
        Console.WriteLine($"Function requested: {call.Name}");
        
        // Execute function
        var result = await ExecuteFunction(call.Name, call.Arguments);
        
        // Create function result
        var resultContent = new FunctionResultContent(call.CallId, result);
        var response = new ExternalInputResponse(
            new ChatMessage(ChatRole.Tool, resultContent)
        );
        
        // Resume workflow with result
    }
}
```

## Approval Workflows

### Simple Approval

```yaml
- kind: InvokeAzureAgent
  id: propose_action
  agent:
    name: PlannerAgent
  output:
    responseObject: Local.ProposedAction

- kind: Question
  id: ask_approval
  prompt: "Approve action: {Local.ProposedAction.description}? (yes/no)"
  property: Local.Approval

- kind: ConditionGroup
  id: check_approval
  conditions:
    - condition: =Lower(Local.Approval) = "yes"
      id: approved
      actions:
        - kind: InvokeAzureAgent
          id: execute_action
          agent:
            name: ExecutorAgent
          input:
            arguments:
              action: =Local.ProposedAction
  
  elseActions:
    - kind: SendActivity
      id: rejected
      activity: "Action rejected by user"
```

### Multi-Level Approval

```yaml
- kind: Question
  id: level1_approval
  prompt: "Manager approval required. Approve? (yes/no)"
  property: Local.ManagerApproval

- kind: ConditionGroup
  id: check_manager
  conditions:
    - condition: =Lower(Local.ManagerApproval) = "yes"
      id: manager_approved
      actions:
        - kind: Question
          id: level2_approval
          prompt: "Director approval required. Approve? (yes/no)"
          property: Local.DirectorApproval
        
        - kind: ConditionGroup
          id: check_director
          conditions:
            - condition: =Lower(Local.DirectorApproval) = "yes"
              id: fully_approved
              actions:
                - kind: SendActivity
                  id: proceed
                  activity: "Fully approved. Proceeding..."
```

## Interactive Troubleshooting

```yaml
# Initialize state
- kind: SetVariable
  id: init_resolved
  variable: Local.IsResolved
  value: false

# Interactive support loop
- kind: InvokeAzureAgent
  id: support_agent
  agent:
    name: SupportAgent
  input:
    externalLoop:
      when: =Not(Local.IsResolved) And Not(Local.NeedsEscalation)
  output:
    autoSend: true
    responseObject: Local.SupportStatus

# Check resolution
- kind: ConditionGroup
  id: check_resolution
  conditions:
    - condition: =Local.SupportStatus.IsResolved
      id: resolved
      actions:
        - kind: SendActivity
          id: success_msg
          activity: "Issue resolved!"
    
    - condition: =Local.SupportStatus.NeedsEscalation
      id: escalate
      actions:
        - kind: SendActivity
          id: escalate_msg
          activity: "Escalating to senior support..."
```

## Tool Approval Pattern

For sensitive operations requiring approval:

```yaml
# Agent proposes to use a tool
- kind: InvokeAzureAgent
  id: agent_with_tools
  agent:
    name: DataAgent
  # Agent may request sensitive tool execution

# ExternalInputRequest emitted with function call
# User approves/rejects the tool call
# Response includes approved function execution results
```

## Best Practices

1. **Clear Prompts** - Make questions specific and actionable
2. **Timeout Handling** - Implement timeouts for user responses
3. **Default Values** - Provide sensible defaults where possible
4. **Validation** - Validate user input before proceeding
5. **Cancellation** - Allow users to cancel operations
6. **Progress Indication** - Show what's waiting for input
7. **Context Preservation** - Maintain conversation context across interactions
8. **Error Messages** - Provide clear feedback for invalid input

## Next Steps

- Learn about [Expressions and PowerFx](./09-expressions-and-powerfx.md)
- Understand [Workflow Execution and Runtime](./10-workflow-execution-and-runtime.md)
- See [Examples and Sample Workflows](./11-examples-and-samples.md)
