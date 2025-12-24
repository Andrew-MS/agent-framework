# Action Types Reference

## Overview

This document provides a comprehensive reference for all action types available in declarative workflows. Actions are the building blocks of workflows and are organized into functional categories.

## Action Categories

1. **Foundry Actions** - Agent and conversation operations
2. **State Management** - Variable and data operations
3. **Control Flow** - Conditional logic and flow control
4. **Human Input** - Interactive user prompts

---

## Foundry Actions

### InvokeAzureAgent

Invokes an Azure AI agent to process a request and return a response.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `InvokeAzureAgent` |
| `id` | String | Yes | Unique action identifier |
| `displayName` | String | No | Human-readable name |
| `conversationId` | Expression | No | Conversation ID (defaults to System.ConversationId) |
| `agent` | Object | Yes | Agent specification |
| `agent.name` | String | Yes | Name of the agent to invoke |
| `agent.version` | String | No | Optional agent version |
| `input` | Object | No | Input configuration |
| `input.messages` | Expression | No | Messages to send to agent |
| `input.arguments` | Object | No | Key-value arguments for the agent |
| `input.externalLoop` | Object | No | External loop configuration |
| `input.externalLoop.when` | Expression | Yes | Condition to continue looping |
| `output` | Object | No | Output configuration |
| `output.autoSend` | Boolean | No | Automatically send response to user |
| `output.messages` | Variable | No | Variable to store response messages |
| `output.responseObject` | Variable | No | Variable to store structured response |

**Example:**

```yaml
- kind: InvokeAzureAgent
  id: research_agent
  displayName: "Gather Research Facts"
  conversationId: =System.ConversationId
  agent:
    name: ResearchAgent
    version: "1.0"
  input:
    messages: =UserMessage(Local.Query)
    arguments:
      context: =Local.AdditionalContext
      depth: "detailed"
  output:
    autoSend: false
    messages: Local.ResearchResponse
    responseObject: Local.ResearchData
```

**Example with External Loop:**

```yaml
- kind: InvokeAzureAgent
  id: support_agent
  agent:
    name: SupportAgent
  input:
    externalLoop:
      when: =Not(Local.IssueResolved) And Not(Local.NeedsEscalation)
  output:
    responseObject: Local.SupportStatus
```

---

### CreateConversation

Creates a new conversation and stores its identifier.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `CreateConversation` |
| `id` | String | Yes | Unique action identifier |
| `conversationId` | Variable | Yes | Variable to store the new conversation ID |

**Example:**

```yaml
- kind: CreateConversation
  id: create_support_conversation
  conversationId: Local.SupportConversationId

# Later use the conversation
- kind: InvokeAzureAgent
  id: invoke_in_conversation
  conversationId: =Local.SupportConversationId
  agent:
    name: MyAgent
```

---

### AddConversationMessage

Adds a message to an existing conversation.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `AddConversationMessage` |
| `id` | String | Yes | Unique action identifier |
| `conversationId` | Expression | Yes | Target conversation ID |
| `message` | Expression | Yes | The message to add |

**Example:**

```yaml
- kind: AddConversationMessage
  id: add_context_message
  conversationId: =Local.ConversationId
  message: =UserMessage("Additional context: " & Local.Context)
```

---

### CopyConversationMessages

Copies messages from one conversation to another.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `CopyConversationMessages` |
| `id` | String | Yes | Unique action identifier |
| `sourceConversationId` | Expression | Yes | Source conversation ID |
| `targetConversationId` | Expression | Yes | Target conversation ID |
| `limit` | Number | No | Maximum number of messages to copy |

**Example:**

```yaml
- kind: CopyConversationMessages
  id: copy_messages
  sourceConversationId: =System.ConversationId
  targetConversationId: =Local.ArchiveConversationId
  limit: 50
```

---

### RetrieveConversationMessage

Retrieves a specific message from a conversation.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `RetrieveConversationMessage` |
| `id` | String | Yes | Unique action identifier |
| `conversationId` | Expression | Yes | Target conversation ID |
| `messageId` | Expression | Yes | ID of the message to retrieve |
| `output` | Variable | Yes | Variable to store the retrieved message |

**Example:**

```yaml
- kind: RetrieveConversationMessage
  id: get_message
  conversationId: =System.ConversationId
  messageId: =Local.StoredMessageId
  output: Local.RetrievedMessage
```

---

### RetrieveConversationMessages

Retrieves multiple messages from a conversation.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `RetrieveConversationMessages` |
| `id` | String | Yes | Unique action identifier |
| `conversationId` | Expression | Yes | Target conversation ID |
| `limit` | Number | No | Maximum number of messages |
| `after` | String | No | Cursor for pagination |
| `before` | String | No | Cursor for pagination |
| `newestFirst` | Boolean | No | Sort order |
| `output` | Variable | Yes | Variable to store messages |

**Example:**

```yaml
- kind: RetrieveConversationMessages
  id: get_history
  conversationId: =System.ConversationId
  limit: 20
  newestFirst: true
  output: Local.ConversationHistory
```

---

### DeleteConversation

Permanently deletes a conversation.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `DeleteConversation` |
| `id` | String | Yes | Unique action identifier |
| `conversationId` | Expression | Yes | ID of conversation to delete |

**Example:**

```yaml
- kind: DeleteConversation
  id: cleanup_conversation
  conversationId: =Local.TempConversationId
```

---

## State Management Actions

### SetVariable

Sets or updates the value of a variable.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `SetVariable` |
| `id` | String | Yes | Unique action identifier |
| `displayName` | String | No | Human-readable name |
| `variable` | Variable | Yes | Variable to set (e.g., Local.Counter) |
| `value` | Expression | Yes | Value or expression to assign |

**Example:**

```yaml
# Literal value
- kind: SetVariable
  id: set_counter
  variable: Local.Counter
  value: 0

# Expression
- kind: SetVariable
  id: increment_counter
  variable: Local.Counter
  value: =Local.Counter + 1

# Complex expression
- kind: SetVariable
  id: set_next_speaker
  variable: Local.NextSpeaker
  value: =Search(Local.Agents, Local.SelectedName, name)

# Array
- kind: SetVariable
  id: set_agents
  variable: Local.AvailableAgents
  value: |-
    =[
      {name: "Agent1", role: "Research"},
      {name: "Agent2", role: "Analysis"}
    ]
```

---

### SetTextVariable

Sets a text variable with string interpolation support.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `SetTextVariable` |
| `id` | String | Yes | Unique action identifier |
| `variable` | Variable | Yes | Variable to set |
| `value` | String | Yes | Text value with {Variable} interpolation |

**Example:**

```yaml
- kind: SetTextVariable
  id: set_instructions
  variable: Local.TaskInstructions
  value: |-
    # TASK
    Address the following user request:
    
    {Local.InputTask}
    
    # TEAM
    Use the following team:
    
    {Local.TeamDescription}
    
    # FACTS
    Consider these facts:
    
    {MessageText(Local.Facts)}
```

---

### SetMultipleVariables

Sets multiple variables in a single action.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `SetMultipleVariables` |
| `id` | String | Yes | Unique action identifier |
| `variables` | Array | Yes | Array of variable assignments |

**Example:**

```yaml
- kind: SetMultipleVariables
  id: initialize_state
  variables:
    - variable: Local.Counter
      value: 0
    - variable: Local.IsComplete
      value: false
    - variable: Local.Status
      value: "Pending"
```

---

### ResetVariable

Resets a variable to its default/initial value.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `ResetVariable` |
| `id` | String | Yes | Unique action identifier |
| `variable` | Variable | Yes | Variable to reset |

**Example:**

```yaml
- kind: ResetVariable
  id: clear_response
  displayName: "Clear Agent Response"
  variable: Local.AgentResponse
```

---

### ClearAllVariables

Clears all local variables.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `ClearAllVariables` |
| `id` | String | Yes | Unique action identifier |

**Example:**

```yaml
- kind: ClearAllVariables
  id: reset_state
```

---

### ParseValue

Parses and converts data from one format to another.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `ParseValue` |
| `id` | String | Yes | Unique action identifier |
| `value` | Expression | Yes | Value to parse |
| `output` | Variable | Yes | Variable to store parsed result |

**Example:**

```yaml
- kind: ParseValue
  id: parse_json
  value: =Local.JsonString
  output: Local.ParsedObject
```

---

### EditTableV2

Modifies data in a structured table/array format.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `EditTableV2` |
| `id` | String | Yes | Unique action identifier |
| `table` | Variable | Yes | Table variable to modify |
| `operation` | String | Yes | Operation type (add, update, delete) |
| `row` | Object | Conditional | Row data for add/update |
| `condition` | Expression | Conditional | Condition for update/delete |

**Example:**

```yaml
- kind: EditTableV2
  id: add_agent
  table: Local.AgentList
  operation: add
  row:
    name: "NewAgent"
    role: "Support"
    status: "Active"
```

---

### SendActivity

Sends an activity message to the user.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `SendActivity` |
| `id` | String | Yes | Unique action identifier |
| `activity` | String | Yes | Message text with {Variable} interpolation |

**Example:**

```yaml
- kind: SendActivity
  id: send_status
  activity: "Processing request... {Local.ProgressPercent}% complete"

- kind: SendActivity
  id: send_greeting
  activity: |
    Welcome to the support system!
    
    Your ticket #{Local.TicketId} has been created.
    Status: {Local.TicketStatus}
```

---

## Control Flow Actions

### ConditionGroup

Evaluates multiple conditions and executes corresponding actions.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `ConditionGroup` |
| `id` | String | Yes | Unique action identifier |
| `conditions` | Array | Yes | Array of condition items |
| `elseActions` | Array | No | Actions to execute if no conditions match |

**Condition Item Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | String | Yes | Unique condition identifier |
| `condition` | Expression | Yes | Boolean expression to evaluate |
| `displayName` | String | No | Human-readable name |
| `actions` | Array | Yes | Actions to execute if condition is true |

**Example:**

```yaml
- kind: ConditionGroup
  id: check_status
  conditions:
    # First condition
    - id: condition_complete
      condition: =Local.Status = "Complete"
      displayName: "When Complete"
      actions:
        - kind: SendActivity
          id: send_success
          activity: "Task completed successfully!"
        - kind: EndWorkflow
          id: end_success
    
    # Second condition
    - id: condition_failed
      condition: =Local.Status = "Failed"
      displayName: "When Failed"
      actions:
        - kind: SendActivity
          id: send_failure
          activity: "Task failed."
        - kind: GotoAction
          id: goto_retry
          actionId: retry_action
    
    # Third condition
    - id: condition_pending
      condition: =Local.Status = "Pending"
      displayName: "When Pending"
      actions:
        - kind: SendActivity
          id: send_waiting
          activity: "Still processing..."
  
  # Else block
  elseActions:
    - kind: SendActivity
      id: send_unknown
      activity: "Unknown status: {Local.Status}"
```

---

### Foreach

Iterates over a collection and executes actions for each item.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `Foreach` |
| `id` | String | Yes | Unique action identifier |
| `itemsProperty` | Expression | Yes | Collection to iterate over |
| `actions` | Array | Yes | Actions to execute for each item |

**Example:**

```yaml
- kind: Foreach
  id: process_agents
  itemsProperty: =Local.AgentList
  actions:
    - kind: SendActivity
      id: send_agent_info
      activity: "Processing agent: {ThisItem.name}"
    
    - kind: InvokeAzureAgent
      id: invoke_agent
      agent:
        name: =ThisItem.name
      input:
        messages: =UserMessage(Local.Task)
```

**Special Variables in Foreach:**
- `ThisItem` - Current item in the iteration
- `ThisItem.property` - Access properties of current item

---

### GotoAction

Jumps to another action in the workflow.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `GotoAction` |
| `id` | String | Yes | Unique action identifier |
| `actionId` | String | Yes | ID of the target action to jump to |

**Example:**

```yaml
- kind: GotoAction
  id: goto_start
  actionId: question_agent

# Create a loop
- kind: SetVariable
  id: increment_counter
  variable: Local.Counter
  value: =Local.Counter + 1

- kind: ConditionGroup
  id: check_counter
  conditions:
    - condition: =Local.Counter < 5
      id: continue_loop
      actions:
        - kind: GotoAction
          id: goto_loop_start
          actionId: increment_counter
```

**Usage Notes:**
- Can create loops by jumping to earlier actions
- Can create conditional branching
- Use with caution to avoid infinite loops

---

### BreakLoop

Exits the current loop immediately.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `BreakLoop` |
| `id` | String | Yes | Unique action identifier |

**Example:**

```yaml
- kind: Foreach
  id: search_items
  itemsProperty: =Local.Items
  actions:
    - kind: ConditionGroup
      id: check_match
      conditions:
        - condition: =ThisItem.value = Local.SearchTarget
          id: found_match
          actions:
            - kind: SetVariable
              id: set_result
              variable: Local.FoundItem
              value: =ThisItem
            - kind: BreakLoop
              id: exit_loop
```

---

### ContinueLoop

Skips the rest of the current iteration and continues with the next.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `ContinueLoop` |
| `id` | String | Yes | Unique action identifier |

**Example:**

```yaml
- kind: Foreach
  id: process_items
  itemsProperty: =Local.Items
  actions:
    # Skip inactive items
    - kind: ConditionGroup
      id: check_active
      conditions:
        - condition: =Not(ThisItem.isActive)
          id: skip_inactive
          actions:
            - kind: ContinueLoop
              id: skip_this
    
    # Process active items
    - kind: SendActivity
      id: process_active
      activity: "Processing: {ThisItem.name}"
```

---

### EndWorkflow

Terminates the workflow execution.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `EndWorkflow` |
| `id` | String | Yes | Unique action identifier |

**Example:**

```yaml
- kind: ConditionGroup
  id: check_completion
  conditions:
    - condition: =Local.IsComplete
      id: if_complete
      actions:
        - kind: SendActivity
          id: send_done
          activity: "All done!"
        - kind: EndWorkflow
          id: end_workflow
```

---

### EndConversation

Ends the current conversation (but may continue the workflow).

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `EndConversation` |
| `id` | String | Yes | Unique action identifier |

**Example:**

```yaml
- kind: EndConversation
  id: end_conversation
```

---

## Human Input Actions

### Question

Presents a question to the user and waits for input.

**Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `kind` | String | Yes | Must be `Question` |
| `id` | String | Yes | Unique action identifier |
| `prompt` | String | Yes | Question text to display |
| `property` | Variable | Yes | Variable to store the response |

**Example:**

```yaml
- kind: Question
  id: ask_name
  prompt: "What is your name?"
  property: Local.UserName

- kind: SendActivity
  id: greet_user
  activity: "Hello, {Local.UserName}!"
```

---

## Action Execution Order

Actions execute sequentially unless:
1. A **ConditionGroup** branches the flow
2. A **GotoAction** jumps to another action
3. A **BreakLoop** or **ContinueLoop** affects iteration
4. An **EndWorkflow** or **EndConversation** terminates execution

## Next Steps

- Learn about [Control Flow Patterns](./07-control-flow-patterns.md)
- Understand [Event Types and Processing](./04-event-types-and-processing.md)
- Explore [State Management and Checkpointing](./05-state-management-and-checkpointing.md)
