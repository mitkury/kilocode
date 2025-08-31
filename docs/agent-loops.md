# Agent Loops in Kilo Code

## Introduction

This document explains how agent loops work in Kilo Code, including model selection, available tools, tool execution flow, and the overall agent loop structure. The agent system is designed to handle complex tasks through iterative tool use and user interaction, with built-in safeguards and optimization features.

## Core Concepts

### What is an Agent Loop?

An agent loop in Kilo Code is the iterative process where:
1. The AI model receives a task and context
2. The model decides which tools to use to accomplish the task
3. Tools are executed and results are returned
4. The model processes results and decides on next actions
5. This continues until task completion or user intervention

The loop ensures that complex tasks can be broken down into manageable steps, with each step building on the results of previous steps.

## Model Selection

### How Models Are Selected

Models in Kilo Code are selected based on several factors:

1. **Provider Configuration**: The system supports multiple AI providers including:
   - Anthropic (Claude models)
   - OpenAI (GPT models)
   - Google (Gemini models)
   - Local models via Ollama
   - Custom MCP servers

2. **Model Capabilities**: Different models have different capabilities:
   - **Function Calling**: Support for tool use via function calling
   - **Context Window**: Maximum token limits for input/output
   - **Cost**: Token pricing and usage limits
   - **Speed**: Response time and streaming capabilities

3. **Task Requirements**: The system may select different models based on:
   - Task complexity
   - Required tool capabilities
   - Cost constraints
   - Performance requirements

### Model Configuration

Models are configured through the API settings in the task:

```typescript
interface ProviderSettings {
  apiProvider: string
  modelId: string
  apiKey?: string
  baseUrl?: string
  // ... other configuration options
}
```

The system automatically handles:
- API key management
- Rate limiting
- Error handling and retries
- Cost tracking and usage monitoring

## Available Tools

### Tool Categories

Tools in Kilo Code are organized into logical groups:

#### 1. Read Tools
- `read_file` - Read file contents with line numbers
- `fetch_instructions` - Fetch instructions from external sources
- `search_files` - Search files using regex patterns
- `list_files` - List directory contents
- `list_code_definition_names` - Find code definitions
- `codebase_search` - Semantic search across codebase

#### 2. Edit Tools
- `apply_diff` - Apply diffs to files
- `edit_file` - AI-powered file editing (Morph FastApply)
- `write_to_file` - Write complete file content
- `insert_content` - Insert content at specific line numbers
- `search_and_replace` - Find and replace text
- `new_rule` - Create new Kilo Code rules

#### 3. Browser Tools
- `browser_action` - Control web browser automation

#### 4. Command Tools
- `execute_command` - Run terminal commands

#### 5. MCP Tools
- `use_mcp_tool` - Use tools from MCP servers
- `access_mcp_resource` - Access resources from MCP servers

#### 6. Mode Tools
- `switch_mode` - Switch between different agent modes
- `new_task` - Create new subtasks

#### 7. Always Available Tools
- `ask_followup_question` - Ask user for clarification
- `attempt_completion` - Signal task completion
- `report_bug` - Report issues
- `condense` - Condense conversation history
- `update_todo_list` - Update task todo list

### Tool Descriptions

Each tool has a detailed description that includes:
- Purpose and functionality
- Required and optional parameters
- Usage examples
- Safety considerations
- Mode-specific availability

Example tool description:
```typescript
export function getReadFileDescription(args: ToolArgs): string {
  return `- **read_file**: Request to read the contents of a file. The tool outputs line-numbered content (e.g. "1 | const x = 1") for easy reference when creating diffs or discussing code.
  
  Parameters:
  - path: The file path to read
  - start_line: (optional) Starting line number
  - end_line: (optional) Ending line number`
}
```

## Tool Execution Flow

### 1. Tool Use Detection

The system parses assistant messages to detect tool use:

```typescript
interface ToolUse {
  type: "tool_use"
  name: ToolName
  params: Partial<Record<ToolParamName, string>>
  partial: boolean
}
```

Tool use is detected through XML-style tags in the assistant's response:
```xml
<read_file>
<path>src/main.ts</path>
</read_file>
```

### 2. Tool Validation

Before execution, tools are validated for:
- **Mode Compatibility**: Tools must be allowed in the current mode
- **Parameter Validation**: Required parameters must be present and valid
- **Safety Checks**: File paths, commands, and other inputs are validated
- **Repetition Detection**: Prevents infinite loops from repeated tool calls

### 3. User Approval

Most tools require user approval before execution:
- **Automatic Approval**: Some tools (like `read_file`) may be auto-approved
- **Manual Approval**: Destructive operations require explicit user consent
- **Batch Approval**: Multiple related operations can be approved together

### 4. Tool Execution

Tools are executed with the following flow:

```typescript
async function executeTool(
  task: Task,
  toolUse: ToolUse,
  askApproval: AskApproval,
  handleError: HandleError,
  pushToolResult: PushToolResult
) {
  // 1. Validate tool use
  validateToolUse(toolUse.name, mode, customModes, experiments, toolUse.params)
  
  // 2. Check for repetition
  const repetitionCheck = task.toolRepetitionDetector.check(toolUse)
  
  // 3. Request user approval
  const approved = await askApproval("tool", toolMessage)
  
  // 4. Execute tool
  try {
    const result = await toolImplementation(toolUse.params)
    pushToolResult(result)
  } catch (error) {
    await handleError("executing tool", error)
  }
}
```

### 5. Result Processing

Tool results are processed and added to the conversation:

```typescript
const pushToolResult = (content: ToolResponse) => {
  task.userMessageContent.push({ 
    type: "text", 
    text: `${toolDescription()} Result:` 
  })
  
  if (typeof content === "string") {
    task.userMessageContent.push({ type: "text", text: content })
  } else {
    task.userMessageContent.push(...content)
  }
}
```

## Agent Loop Structure

### Main Loop Flow

The agent loop follows this structure:

```typescript
async function initiateTaskLoop(userContent: ContentBlock[]) {
  while (!task.abort) {
    // 1. Make API request to model
    const didEndLoop = await recursivelyMakeClineRequests(userContent, includeFileDetails)
    
    // 2. Check if loop should end
    if (didEndLoop) {
      break
    } else {
      // 3. Continue with next iteration
      userContent = [{ type: "text", text: formatResponse.noToolsUsed() }]
      task.consecutiveMistakeCount++
    }
  }
}
```

### Recursive Request Processing

The core loop handles multiple requests per task:

```typescript
async function recursivelyMakeClineRequests(
  userContent: ContentBlock[],
  includeFileDetails: boolean = false
): Promise<boolean> {
  const stack: StackItem[] = [{ userContent, includeFileDetails }]
  
  while (stack.length > 0) {
    const currentItem = stack.pop()!
    
    // 1. Check for abort conditions
    if (task.abort) {
      throw new Error("Task aborted")
    }
    
    // 2. Check mistake limits
    if (task.consecutiveMistakeCount >= task.consecutiveMistakeLimit) {
      // Handle mistake limit reached
    }
    
    // 3. Check for paused state
    if (task.isPaused) {
      await task.waitForResume()
    }
    
    // 4. Prepare environment details
    const environmentDetails = await getEnvironmentDetails(task, includeFileDetails)
    
    // 5. Make API request
    const stream = task.attemptApiRequest()
    
    // 6. Process streaming response
    for await (const chunk of stream) {
      switch (chunk.type) {
        case "reasoning":
          // Handle reasoning chunks
          break
        case "text":
          // Parse and present text content
          break
        case "usage":
          // Track token usage
          break
      }
    }
    
    // 7. Process tool use
    if (task.userMessageContent.length > 0) {
      stack.push({
        userContent: [...task.userMessageContent],
        includeFileDetails: false
      })
    }
  }
  
  return false // Continue loop
}
```

### Streaming Response Processing

The system processes streaming responses in real-time:

```typescript
// Parse raw assistant message chunk into content blocks
const prevLength = task.assistantMessageContent.length
task.assistantMessageContent = task.assistantMessageParser.processChunk(chunk.text)

if (task.assistantMessageContent.length > prevLength) {
  // New content to present
  task.userMessageContentReady = false
}

// Present content to user
presentAssistantMessage(task)
```

### Tool Execution in Loop

Tools are executed as part of the message presentation:

```typescript
async function presentAssistantMessage(task: Task) {
  const block = task.assistantMessageContent[task.currentStreamingContentIndex]
  
  switch (block.type) {
    case "text":
      // Present text content
      await task.say("text", content, undefined, block.partial)
      break
      
    case "tool_use":
      // Execute tool
      await executeToolImplementation(task, block)
      break
  }
  
  // Move to next content block
  task.currentStreamingContentIndex++
}
```

## Safety and Control Mechanisms

### 1. Mistake Detection

The system tracks consecutive mistakes to prevent infinite loops:

```typescript
class ToolRepetitionDetector {
  check(toolUse: ToolUse): RepetitionCheck {
    // Check for identical consecutive tool calls
    // Return whether execution should be allowed
  }
}
```

### 2. User Control

Users can:
- **Pause/Resume**: Temporarily stop the agent
- **Cancel**: Abort the entire task
- **Reject Tools**: Decline specific tool executions
- **Provide Feedback**: Give guidance during execution

### 3. Error Handling

Comprehensive error handling includes:
- **API Errors**: Rate limits, network issues, model errors
- **Tool Errors**: File not found, permission denied, invalid parameters
- **System Errors**: Memory issues, disk space, configuration problems

### 4. Checkpointing

The system saves checkpoints to allow task resumption:

```typescript
async function checkpointSaveAndMark(task: Task) {
  if (!task.currentStreamingDidCheckpoint) {
    await task.checkpointSave(true)
    task.currentStreamingDidCheckpoint = true
  }
}
```

## Mode System

### Available Modes

Different modes provide different tool sets and behaviors:

1. **Architect Mode**: Planning and high-level design
2. **Coder Mode**: Implementation and code generation
3. **Debugger Mode**: Problem-solving and debugging
4. **Custom Modes**: User-defined modes with specific capabilities

### Mode Switching

Modes can be switched during task execution:

```typescript
case "switch_mode":
  await switchModeTool(task, block, askApproval, handleError, pushToolResult, removeClosingTag)
  break
```

## Performance Optimizations

### 1. Streaming

Responses are streamed in real-time for better UX:
- Immediate feedback to users
- Progressive content display
- Early tool execution detection

### 2. Caching

Token usage and API responses are cached:
- Reduce API costs
- Improve response times
- Handle rate limits gracefully

### 3. Background Processing

Heavy operations run in background:
- File system operations
- Code indexing
- Usage data collection

### 4. Memory Management

Efficient memory usage through:
- Streaming processing
- Garbage collection
- Memory limits and cleanup

## Conclusion

The agent loop system in Kilo Code provides a robust, safe, and efficient way to handle complex AI-powered tasks. Through careful tool design, comprehensive safety mechanisms, and intelligent loop management, the system can handle a wide variety of tasks while maintaining user control and system stability.

The modular design allows for easy extension with new tools, modes, and capabilities while maintaining the core loop structure that ensures reliable task execution.