# Examples and Sample Workflows

## Overview

This document provides complete examples of declarative workflows, from simple to complex patterns.

## Simple Examples

### Hello World

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: hello_workflow
  actions:
    - kind: SendActivity
      id: send_hello
      activity: "Hello, World!"
```

### Echo Workflow

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: echo_workflow
  actions:
    - kind: SendActivity
      id: echo
      activity: "You said: {System.LastMessage.Text}"
```

### Counter Workflow

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: counter_workflow
  actions:
    - kind: SetVariable
      id: init
      variable: Local.Count
      value: 0
    
    - kind: SetVariable
      id: increment
      variable: Local.Count
      value: =Local.Count + 1
    
    - kind: SendActivity
      id: show
      activity: "Count: {Local.Count}"
```

## Intermediate Examples

### Simple Agent Interaction

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: simple_agent
  actions:
    - kind: InvokeAzureAgent
      id: invoke_agent
      conversationId: =System.ConversationId
      agent:
        name: AssistantAgent
      output:
        autoSend: true
```

### Agent with Variable Storage

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: agent_with_storage
  actions:
    - kind: InvokeAzureAgent
      id: research_agent
      agent:
        name: ResearchAgent
      input:
        messages: =UserMessage(System.LastMessage.Text)
      output:
        messages: Local.ResearchResults
    
    - kind: SendActivity
      id: summary
      activity: "Research complete. Found {CountRows(Local.ResearchResults)} results."
```

### Conditional Routing

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: conditional_routing
  actions:
    # Classify the request
    - kind: InvokeAzureAgent
      id: classifier
      agent:
        name: ClassifierAgent
      output:
        responseObject: Local.Classification
    
    # Route based on classification
    - kind: ConditionGroup
      id: route
      conditions:
        - condition: =Local.Classification.type = "Technical"
          id: route_tech
          actions:
            - kind: SendActivity
              id: notify_tech
              activity: "Routing to technical support..."
            
            - kind: InvokeAzureAgent
              id: tech_agent
              agent:
                name: TechnicalAgent
              output:
                autoSend: true
        
        - condition: =Local.Classification.type = "Billing"
          id: route_billing
          actions:
            - kind: SendActivity
              id: notify_billing
              activity: "Routing to billing department..."
            
            - kind: InvokeAzureAgent
              id: billing_agent
              agent:
                name: BillingAgent
              output:
                autoSend: true
      
      elseActions:
        - kind: SendActivity
          id: route_general
          activity: "Routing to general support..."
```

## Advanced Examples

### Multi-Agent Research Pipeline

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: research_pipeline
  actions:
    # Get user's research topic
    - kind: SetVariable
      id: set_topic
      variable: Local.Topic
      value: =System.LastMessage.Text
    
    # Stage 1: Gather facts
    - kind: SendActivity
      id: stage1_start
      activity: "Gathering facts about: {Local.Topic}"
    
    - kind: InvokeAzureAgent
      id: fact_gatherer
      agent:
        name: FactGathererAgent
      input:
        messages: =UserMessage(Local.Topic)
      output:
        responseObject: Local.Facts
    
    # Stage 2: Analyze data
    - kind: SendActivity
      id: stage2_start
      activity: "Analyzing data..."
    
    - kind: InvokeAzureAgent
      id: analyzer
      conversationId: =System.ConversationId
      agent:
        name: AnalyzerAgent
      input:
        arguments:
          facts: =Local.Facts
      output:
        responseObject: Local.Analysis
    
    # Stage 3: Generate report
    - kind: SendActivity
      id: stage3_start
      activity: "Generating report..."
    
    - kind: InvokeAzureAgent
      id: reporter
      conversationId: =System.ConversationId
      agent:
        name: ReporterAgent
      input:
        arguments:
          facts: =Local.Facts
          analysis: =Local.Analysis
      output:
        autoSend: true
```

### Interactive Troubleshooting

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: troubleshooting_workflow
  actions:
    # Initialize
    - kind: SetVariable
      id: init_resolved
      variable: Local.IsResolved
      value: false
    
    - kind: SetVariable
      id: init_attempts
      variable: Local.AttemptCount
      value: 0
    
    # Interactive loop
    - kind: InvokeAzureAgent
      id: support_agent
      conversationId: =System.ConversationId
      agent:
        name: SupportAgent
      input:
        externalLoop:
          when: |-
            =Not(Local.IsResolved) 
             And Local.AttemptCount < 5
      output:
        autoSend: true
        responseObject: Local.SupportStatus
    
    # Update attempt counter
    - kind: SetVariable
      id: increment_attempts
      variable: Local.AttemptCount
      value: =Local.AttemptCount + 1
    
    # Check resolution
    - kind: ConditionGroup
      id: check_resolution
      conditions:
        - condition: =Local.SupportStatus.IsResolved
          id: resolved
          actions:
            - kind: SendActivity
              id: success
              activity: "Issue resolved successfully!"
            
            - kind: EndWorkflow
              id: end_success
        
        - condition: =Local.AttemptCount >= 5
          id: max_attempts
          actions:
            - kind: SendActivity
              id: escalate
              activity: "Unable to resolve. Escalating to senior support..."
```

### Approval Workflow

```yaml
kind: Workflow
trigger:
  kind: OnConversationStart
  id: approval_workflow
  actions:
    # Get request details
    - kind: InvokeAzureAgent
      id: parse_request
      agent:
        name: RequestParserAgent
      output:
        responseObject: Local.Request
    
    # Check amount threshold
    - kind: ConditionGroup
      id: check_amount
      conditions:
        - condition: =Local.Request.amount > 10000
          id: high_amount
          actions:
            # Require senior approval
            - kind: Question
              id: senior_approval
              prompt: "Senior approval required for ${Local.Request.amount}. Approve? (yes/no)"
              property: Local.SeniorApproval
            
            - kind: ConditionGroup
              id: check_senior
              conditions:
                - condition: =Lower(Local.SeniorApproval) <> "yes"
                  id: senior_rejected
                  actions:
                    - kind: SendActivity
                      id: rejected_msg
                      activity: "Request rejected by senior approver"
                    
                    - kind: EndWorkflow
                      id: end_rejected
      
      # Always require manager approval
      - kind: Question
        id: manager_approval
        prompt: "Manager approval for {Local.Request.description}. Approve? (yes/no)"
        property: Local.ManagerApproval
      
      - kind: ConditionGroup
        id: check_manager
        conditions:
          - condition: =Lower(Local.ManagerApproval) = "yes"
            id: approved
            actions:
              - kind: SendActivity
                id: approved_msg
                activity: "Request approved! Processing..."
              
              - kind: InvokeAzureAgent
                id: process_request
                agent:
                  name: RequestProcessorAgent
                input:
                  arguments:
                    request: =Local.Request
                output:
                  autoSend: true
        
        elseActions:
          - kind: SendActivity
            id: denied_msg
            activity: "Request denied by manager"
```

## Complex Workflow: Customer Support System

See the complete [CustomerSupport.yaml](../../workflow-samples/CustomerSupport.yaml) sample for a production-ready customer support workflow featuring:

- Interactive self-service
- Ticket creation
- Automatic routing
- Multi-level support escalation
- Resolution tracking

## Complex Workflow: Deep Research

See the complete [DeepResearch.yaml](../../workflow-samples/DeepResearch.yaml) sample for a Magentic-style orchestration with:

- Multiple specialized agents
- Dynamic planning
- Progress tracking
- Stall detection and recovery
- Fact gathering and analysis

## Complex Workflow: Student-Teacher

See the complete [MathChat.yaml](../../workflow-samples/MathChat.yaml) sample for an educational pattern with:

- Turn-based conversation
- Progress tracking
- Completion detection
- Iterative refinement

## Running Samples

### Setup

1. Install package:
   ```bash
   dotnet add package Microsoft.Agents.AI.Workflows.Declarative
   ```

2. Configure credentials:
   ```bash
   dotnet user-secrets set "FOUNDRY_PROJECT_ENDPOINT" "https://..."
   dotnet user-secrets set "FOUNDRY_MODEL_DEPLOYMENT_NAME" "gpt-4"
   ```

3. Run sample:
   ```bash
   cd dotnet/samples/GettingStarted/Workflows/Declarative/ExecuteWorkflow
   dotnet run workflow.yaml
   ```

### Sample Projects

- **[ExecuteWorkflow](../../dotnet/samples/GettingStarted/Workflows/Declarative/ExecuteWorkflow/)** - Command-line workflow executor
- **[HostedWorkflow](../../dotnet/samples/GettingStarted/Workflows/Declarative/HostedWorkflow/)** - Web-hosted workflow service

## Additional Resources

- [Workflow Samples Directory](../../workflow-samples/) - More YAML samples
- [.NET Workflow Samples](../../dotnet/samples/GettingStarted/Workflows/) - Code-based examples
- [Python Workflow Samples](../../python/samples/getting_started/workflows/) - Python equivalents

## Next Steps

- Review [Best Practices and Patterns](./12-best-practices-and-patterns.md)
- Explore [Agent Providers and Integration](./06-agent-providers-and-integration.md)
- Study complete samples in the workflow-samples directory
