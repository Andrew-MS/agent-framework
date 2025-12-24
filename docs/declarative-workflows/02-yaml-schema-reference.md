# YAML Schema Reference

## Overview

Declarative workflows are defined using YAML files that follow the Bot.ObjectModel schema. This document provides a comprehensive reference for the structure and syntax of workflow YAML files.

## Basic Workflow Structure

Every workflow YAML file follows this basic structure:

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: workflow_identifier
  actions:
    - kind: ActionType
      id: action_id
      # Action-specific properties
    - kind: AnotherActionType
      id: another_action_id
      # More actions...
```

## Root-Level Properties

### kind (Required)

Specifies the type of definition. For workflows, this must always be `Workflow`.

```yaml
kind: Workflow
```

**Type:** String  
**Values:** `Workflow`  
**Required:** Yes

### trigger (Required)

Defines when and how the workflow is initiated.

```yaml
trigger:
  kind: OnConversationStart
  id: my_workflow
  actions:
    # List of actions to execute
```

**Type:** Object  
**Required:** Yes

## Trigger Definition

### trigger.kind

Specifies the trigger type for the workflow.

```yaml
trigger:
  kind: OnConversationStart
```

**Type:** String  
**Values:** Currently `OnConversationStart` is the primary trigger type  
**Required:** Yes

### trigger.id

Unique identifier for the workflow trigger.

```yaml
trigger:
  id: my_workflow_trigger
```

**Type:** String  
**Required:** Yes  
**Guidelines:**
- Use descriptive, snake_case identifiers
- Must be unique within the workflow
- Used for debugging and telemetry

### trigger.actions

Array of actions to execute when the trigger fires.

```yaml
trigger:
  actions:
    - kind: SetVariable
      id: set_counter
      variable: Local.Counter
      value: 0
    - kind: SendActivity
      id: send_greeting
      activity: "Hello, World!"
```

**Type:** Array  
**Required:** Yes  
**Items:** Action objects (see Action Types section)

## Action Common Properties

All actions share these common properties:

### kind (Required)

Specifies the type of action.

```yaml
- kind: InvokeAzureAgent
  # or: SetVariable, ConditionGroup, SendActivity, etc.
```

**Type:** String  
**Required:** Yes  
**See Also:** [Action Types Reference](./03-action-types-reference.md)

### id (Required)

Unique identifier for the action within the workflow.

```yaml
- kind: SetVariable
  id: set_initial_count
```

**Type:** String  
**Required:** Yes  
**Guidelines:**
- Must be unique within the workflow
- Used for GotoAction targeting
- Used in event emission for tracking
- Use descriptive, snake_case names

### displayName (Optional)

Human-readable name for the action, used for documentation and debugging.

```yaml
- kind: InvokeAzureAgent
  id: question_agent_ABC123
  displayName: "Ask the Research Agent for Facts"
```

**Type:** String  
**Required:** No  
**Purpose:** Documentation, debugging, UI display

## Variable References

Variables are referenced using specific prefixes:

### System Variables

System-provided variables that are automatically available:

```yaml
# Access the current conversation ID
conversationId: =System.ConversationId

# Access the last user message
value: =System.LastMessage.Text
```

**Available System Variables:**
- `System.ConversationId` - Current conversation identifier
- `System.LastMessage` - Most recent input message
- `System.LastMessage.Text` - Text content of last message

### Local Variables

User-defined variables stored in the workflow's state:

```yaml
# Define a local variable
- kind: SetVariable
  id: set_counter
  variable: Local.Counter
  value: 0

# Reference the variable
- kind: SendActivity
  id: show_counter
  activity: "Count: {Local.Counter}"
```

**Syntax:** `Local.VariableName`  
**Scope:** Lifetime of the workflow run  
**Persistence:** Maintained in checkpoints

## Expression Syntax

Expressions are PowerFx formulas prefixed with `=`:

### Simple Value Assignment

```yaml
# Literal value
value: 0

# Expression
value: =Local.Counter + 1
```

### String Interpolation

Use curly braces for variable interpolation in strings:

```yaml
# String interpolation (no = prefix)
activity: "Hello, {Local.UserName}!"

# Expression with string concatenation
value: =$"The count is {Local.Counter}"
```

### Boolean Expressions

```yaml
# Conditional expression
condition: =Local.Counter > 5

# Logical operators
condition: =Local.IsValid And Not(Local.IsProcessed)

# Comparison
condition: =Local.Status = "Complete"
```

### Function Calls

```yaml
# Built-in PowerFx functions
value: =Concat(Local.Items, Value, ", ")

# Custom functions
value: =UserMessage(Local.InputText)

# Nested functions
value: =Upper(MessageText(Local.Response))
```

### Collection Operations

```yaml
# Search in array
value: =Search(Local.Agents, "AgentName", name)

# Count items
value: =CountRows(Local.Results)

# Filter
value: =Filter(Local.Items, Status = "Active")

# ForAll (map)
value: =ForAll(Local.Items, Upper(Name))
```

## Complex Property Types

### Conversation ID Reference

Many actions accept a conversation ID:

```yaml
# Use system conversation
conversationId: =System.ConversationId

# Use a stored conversation ID
conversationId: =Local.MyConversationId

# Create new conversation and store ID
- kind: CreateConversation
  id: create_conv
  conversationId: Local.NewConversationId
```

### Agent Definition

Agent references specify which agent to invoke:

```yaml
agent:
  name: ResearchAgent
  # Optional version
  version: "1.0"
```

### Input and Output Configuration

Actions that interact with agents support input/output configuration:

```yaml
- kind: InvokeAzureAgent
  id: invoke_agent
  agent:
    name: MyAgent
  input:
    # Input messages
    messages: =UserMessage(Local.Query)
    
    # Input arguments (key-value pairs)
    arguments:
      team: =Local.TeamDescription
      context: =Local.AdditionalContext
    
    # External loop condition
    externalLoop:
      when: =Not(Local.IsComplete)
      
  output:
    # Auto-send response to user
    autoSend: true
    
    # Store messages in variable
    messages: Local.AgentResponse
    
    # Store structured response
    responseObject: Local.AgentOutput
```

### Condition Groups

Conditions support multiple branches with else fallback:

```yaml
- kind: ConditionGroup
  id: check_status
  conditions:
    # First condition
    - id: condition_success
      condition: =Local.Status = "Success"
      displayName: "When Successful"
      actions:
        - kind: SendActivity
          id: send_success
          activity: "Operation succeeded!"
    
    # Second condition
    - id: condition_failure
      condition: =Local.Status = "Failure"
      displayName: "When Failed"
      actions:
        - kind: SendActivity
          id: send_failure
          activity: "Operation failed."
  
  # Else actions (optional)
  elseActions:
    - kind: SendActivity
      id: send_unknown
      activity: "Unknown status"
```

## Data Types

### Scalars

```yaml
# String
variable: Local.Name
value: "John Doe"

# Number
variable: Local.Count
value: 42

# Boolean
variable: Local.IsEnabled
value: true

# Null
variable: Local.OptionalValue
value: null
```

### Arrays

```yaml
# Literal array
variable: Local.Items
value: |-
  =[
    "Item 1",
    "Item 2",
    "Item 3"
  ]

# Array of objects
variable: Local.Agents
value: |-
  =[
    {name: "Agent1", role: "Research"},
    {name: "Agent2", role: "Analysis"}
  ]
```

### Objects

```yaml
# Object literal
variable: Local.Config
value: |-
  ={
    setting1: "value1",
    setting2: 42,
    nested: {
      key: "value"
    }
  }
```

## Multi-line Strings

YAML supports various multi-line string formats:

### Literal Block Scalar (|)

Preserves line breaks:

```yaml
- kind: SendActivity
  id: send_message
  activity: |
    This is line 1
    This is line 2
    This is line 3
```

### Folded Block Scalar (>)

Folds line breaks into spaces:

```yaml
- kind: SetTextVariable
  id: set_long_text
  variable: Local.Description
  value: >
    This is a long description
    that spans multiple lines
    but will be folded into a single line.
```

### Literal Block Scalar with Strip (|-)

Strips final line breaks:

```yaml
- kind: SetTextVariable
  id: set_text
  variable: Local.Prompt
  value: |-
    This text will have
    no trailing newline
```

## Comments

YAML supports single-line comments:

```yaml
# This is a comment
- kind: SetVariable
  id: set_value
  variable: Local.Counter
  value: 0  # Inline comment
```

## Special Character Escaping

### Curly Braces in Strings

If you need literal curly braces in interpolated strings:

```yaml
# This will interpolate the variable
activity: "Value: {Local.Count}"

# For literal braces, double them
activity: "Use {{braces}} like this"
```

### Quotes in Strings

```yaml
# Single quotes in double-quoted strings
activity: "She said 'hello'"

# Double quotes in single-quoted strings
activity: 'He said "goodbye"'

# Escaped quotes
activity: "She said \"hello\""
```

## Schema Validation

The workflow YAML must conform to the Bot.ObjectModel schema. Common validation errors:

### Missing Required Properties

```yaml
# ❌ Invalid - missing 'kind'
trigger:
  id: my_trigger
  actions: []

# ✅ Valid
trigger:
  kind: OnConversationStart
  id: my_trigger
  actions: []
```

### Invalid Action Structure

```yaml
# ❌ Invalid - missing 'id'
- kind: SetVariable
  variable: Local.Count
  value: 0

# ✅ Valid
- kind: SetVariable
  id: set_count
  variable: Local.Count
  value: 0
```

### Duplicate IDs

```yaml
# ❌ Invalid - duplicate action IDs
- kind: SetVariable
  id: my_action
  variable: Local.A
  value: 1

- kind: SetVariable
  id: my_action  # Duplicate!
  variable: Local.B
  value: 2
```

## Complete Example

Here's a complete, well-structured workflow:

```yaml
#
# Customer Support Workflow
# Demonstrates multi-agent orchestration with conditional routing
#
kind: Workflow
trigger:
  kind: OnConversationStart
  id: customer_support_workflow
  actions:
    
    # Initialize variables
    - kind: SetVariable
      id: initialize_counter
      displayName: "Initialize Attempt Counter"
      variable: Local.AttemptCount
      value: 0
    
    # Get user's issue
    - kind: InvokeAzureAgent
      id: get_issue
      displayName: "Understand Customer Issue"
      conversationId: =System.ConversationId
      agent:
        name: TechSupportAgent
      input:
        messages: =System.LastMessage
      output:
        messages: Local.IssueAnalysis
        responseObject: Local.IssueDetails
    
    # Route based on severity
    - kind: ConditionGroup
      id: route_by_severity
      displayName: "Route Based on Issue Severity"
      conditions:
        
        - id: high_severity
          condition: =Local.IssueDetails.Severity = "High"
          displayName: "High Severity Issues"
          actions:
            - kind: SendActivity
              id: escalate_message
              activity: "Escalating to senior support..."
            
            - kind: GotoAction
              id: goto_escalation
              actionId: escalate_to_senior
        
        - id: medium_severity
          condition: =Local.IssueDetails.Severity = "Medium"
          displayName: "Medium Severity Issues"
          actions:
            - kind: SendActivity
              id: standard_message
              activity: "Processing with standard support..."
      
      elseActions:
        - kind: SendActivity
          id: low_severity_message
          activity: "This looks like a simple issue. Let me help!"
    
    # Continue workflow...
    - kind: EndWorkflow
      id: workflow_complete
    
    # Escalation path
    - kind: InvokeAzureAgent
      id: escalate_to_senior
      displayName: "Escalate to Senior Support"
      agent:
        name: SeniorSupportAgent
      input:
        arguments:
          issue: =Local.IssueDetails
      output:
        autoSend: true
    
    - kind: EndWorkflow
      id: escalation_complete
```

## Best Practices

1. **Use Descriptive IDs**: Choose meaningful action IDs that describe the purpose
2. **Add Display Names**: Include displayName for complex actions
3. **Comment Your Workflow**: Add comments to explain complex logic
4. **Organize Actions**: Group related actions together
5. **Use Variables**: Store reusable values in variables
6. **Validate Expressions**: Test PowerFx expressions before deployment
7. **Handle Errors**: Include error handling paths in conditions
8. **Keep It Readable**: Use consistent indentation (2 or 4 spaces)

## Next Steps

- Explore [Action Types Reference](./03-action-types-reference.md) for detailed action documentation
- Learn about [Expressions and PowerFx](./09-expressions-and-powerfx.md) for advanced formulas
- See [Examples and Sample Workflows](./11-examples-and-samples.md) for complete examples
