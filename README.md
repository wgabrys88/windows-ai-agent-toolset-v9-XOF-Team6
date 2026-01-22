# TECHNICAL ANALYSIS REPORT: AI AGENTIC COMPUTER CONTROL SYSTEM

## SYSTEM CLASSIFICATION

Autonomous AI agent system for executing computer tasks through vision-based UI interaction. Implements dual-agent architecture with stateless LLM communication via OpenAI-compatible API.

## ARCHITECTURE OVERVIEW

### Core Design Pattern
Two-agent collaborative system with mode switching:
- **Planner Agent**: Strategic planning, task decomposition, memory management
- **Executor Agent**: Tactical execution, screenshot analysis, tool invocation

### Execution Flow
```
User Task Input → Initial Planning → Execution Loop → Periodic Review → Task Completion
                       ↓                    ↓              ↓
                   Plan Creation    Tool Execution   Plan Adjustment
                                    Screenshot Analysis
                                    Memory Compression
```

### State Management
Centralized state object (`AgentState`) maintains:
- Current task description
- Screenshot buffer (PNG bytes)
- Screen dimensions (physical pixels)
- Turn counter
- Operating mode (PLANNING/EXECUTION)
- Memory system reference
- Current plan text
- Execution instructions for executor
- Review flag

## COMPONENT BREAKDOWN

### 1. CONFIGURATION SYSTEM

**Class**: `Config` (frozen dataclass)

**Purpose**: Immutable system parameters

**Critical Parameters**:
- `lmstudio_endpoint`: HTTP endpoint for LLM API calls
- `lmstudio_model`: Model identifier string
- `lmstudio_timeout`: 240 seconds for API calls
- `lmstudio_temperature`: 0.5 for executor, 0.35 for planner
- `planner_max_tokens`: 1800 tokens maximum response
- `executor_max_tokens`: 1400 tokens maximum response
- `screen_capture_w/h`: 1536x864 downscaled resolution
- `max_steps`: 50 turn hard limit
- `ui_settle_delay`: 0.3s for UI state stabilization
- `turn_delay`: 1.5s between agent turns
- `char_input_delay`: 0.01s between character inputs
- `review_interval`: 7 turns between forced planning reviews
- `memory_compression_threshold`: 12 active actions before compression required
- `loop_recovery_cooldown`: 3 turns between recovery attempts

**Notes**: All timing parameters calibrated for Windows 11 UI responsiveness and model processing speed constraints.

### 2. EXECUTION LOGGER

**Class**: `ExecutionLogger`

**Responsibilities**:
- Structured logging to `dumps/execution_log.txt`
- Screenshot persistence to `dumps/screenshots/turn_NNNN.png`
- API request/response logging with base64 redaction
- Tool execution tracking
- State change documentation

**Redaction Mechanism**: Replaces base64-encoded image data with length metadata to maintain log readability.

**API Call Counter**: Sequential numbering for request-response correlation.

### 3. WINDOWS API INTEGRATION LAYER

**Implementation**: Direct ctypes bindings to user32.dll and gdi32.dll

**Functions Provided**:

#### Screen Capture Pipeline
- `init_dpi()`: Sets DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 for accurate coordinate mapping with 125% scaling
- `get_screen_size()`: Retrieves SM_CXSCREEN/SM_CYSCREEN metrics
- `capture_png(tw, th)`: 
  - Creates device context (DC) for screen
  - Creates compatible memory DC
  - Creates 32-bit DIB section for pixel buffer
  - Performs StretchBlt with HALFTONE mode for downscaling
  - Draws cursor overlay using GetCursorInfo/GetIconInfo/DrawIconEx
  - Converts BGRA to RGB
  - Compresses to PNG using custom implementation
  - Returns: (png_bytes, physical_screen_width, physical_screen_height)

#### PNG Encoding
- `png_pack(tag, data)`: Creates PNG chunk with CRC32 checksum
- `rgb_to_png(rgb, w, h)`: 
  - Adds filter byte (0x00) to each scanline
  - Compresses with zlib level 6
  - Constructs PNG file structure (signature, IHDR, IDAT, IEND)

#### Input Simulation
- `mouse_move(x, y)`: SetCursorPos with settle delay
- `mouse_click(button)`: SendInput with MOUSEEVENTF_LEFTDOWN/UP or RIGHTDOWN/UP
- `mouse_double_click()`: Two sequential left clicks
- `mouse_drag(x1, y1, x2, y2)`: 15-step interpolated movement with button hold
- `mouse_scroll(direction)`: MOUSEEVENTF_WHEEL with ±120 delta
- `keyboard_type_text(text)`: Unicode keyboard events batched in 100-event chunks
- `keyboard_press_keys(combo)`: Virtual key code mapping with modifier support

**VK_MAP Dictionary**: 
- Alphanumeric: a-z, 0-9
- Function keys: f1-f12
- Special keys: enter, tab, escape, windows, ctrl, alt, shift, backspace, delete, space, home, end, pageup, pagedown, arrow keys

**Critical Note**: All input functions include `ui_settle_delay` waits for UI state stabilization.

### 4. COORDINATE SYSTEM

**Class**: `Coordinate` (frozen dataclass)

**Normalization**: 0-1000 grid in both axes

**Conversion Logic**:
```
normalized_x = clamp(x, 0, 1000)
normalized_y = clamp(y, 0, 1000)
pixel_x = round((normalized_x / 1000.0) * screen_width)
pixel_y = round((normalized_y / 1000.0) * screen_height)
```

**Rationale**: Model-friendly coordinate system independent of physical screen resolution. Reduces coordinate ambiguity for vision model.

### 5. TOOL ARGUMENT SCHEMAS

**Frozen Dataclasses**:
- `ClickArgs`: reasoning, element_name, position(Coordinate)
- `DragArgs`: reasoning, element_name, start(Coordinate), end(Coordinate)
- `TypeTextArgs`: reasoning, text
- `PressKeyArgs`: reasoning, key
- `ScrollArgs`: reasoning
- `ReportCompletionArgs`: evidence
- `ReportProgressArgs`: goal_identifier, completion_status, evidence
- `ArchiveHistoryArgs`: summary, patterns_detected, archived_turns(tuple[int])
- `UpdatePlanArgs`: instructions, reasoning

**Design Pattern**: Frozen dataclasses ensure immutability post-parsing. All tool arguments require explicit reasoning field for model accountability.

### 6. TOOL CALL PARSER

**Class**: `ToolCall`

**Parsing Logic**:
- Extracts function name from `tool_call_dict["function"]["name"]`
- Handles both string-serialized JSON and direct dict formats for arguments
- Maps tool names to argument parser lambdas
- Coordinate arrays converted to Coordinate objects
- Raises ValueError for unknown tools

**Supported Tools**:

#### Planner Tools (2):
- `update_execution_plan`: Strategic instruction delivery
- `archive_completed_actions`: Memory compression

#### Executor Tools (11):
- `report_mission_complete`: Terminal action
- `report_goal_status`: Progress reporting
- `click_screen_element`: Left click
- `double_click_screen_element`: Double left click
- `right_click_screen_element`: Context menu
- `drag_screen_element`: Click-drag operation
- `type_text_input`: Unicode text input
- `press_keyboard_key`: Key/combo press
- `scroll_screen_down`: Downward scroll
- `scroll_screen_up`: Upward scroll

### 7. MEMORY SYSTEM

**Class**: `AgentMemory`

**Data Structures**:
- `_history`: List of ActionRecord objects (mutable archived flag)
- `_snapshots`: List of MemorySnapshot objects (compressed history summaries)
- `last_recovery_turn`: Integer tracking last loop recovery attempt

**ActionRecord Dataclass**:
- turn: int
- tool: str
- args: Any (parsed tool arguments)
- result: str (execution result message)
- screenshot: str (file path)
- response_text: str (LLM textual response)
- archived: bool (compression flag)

**MemorySnapshot Dataclass** (frozen):
- summary: str
- patterns: str
- archived_count: int

**Key Methods**:

- `add_action(record)`: Appends to history
- `active_history`: Property returning tuple of non-archived records
- `all_snapshots`: Property returning tuple of snapshots
- `apply_compression(summary, patterns, archived_turns)`: Sets archived flag on specified turns, creates snapshot
- `get_context_for_llm()`: Formats memory for LLM context
  - Lists snapshots with truncated summaries (180 chars) and patterns (120 chars)
  - Shows last 8 active actions with turn number, tool name, result (60 chars), response (140 chars)
- `get_last_action_for_validation()`: Returns most recent active action for executor validation
- `needs_compression`: Property checking if active history >= threshold (12)
- `mark_recovery(turn)`: Updates last_recovery_turn
- `should_recover(current_turn)`: Checks cooldown period (3 turns)

**Memory Context Format**:
```
COMPLETED_WORK:
  Archive_N: summary...
    Patterns: patterns...

RECENT_ACTIONS (count=X):
  Turn_N: tool_name -> result...
    Response: response_text...
```

**Compression Strategy**: Marks old actions as archived without deletion. Allows historical reference while reducing token consumption.

### 8. AGENT STATE

**Class**: `AgentState` (dataclass)

**Fields**:
- `task`: str (immutable user task)
- `screenshot`: Optional[bytes] (current PNG buffer)
- `screen_dims`: tuple[int, int] (physical width, height)
- `turn`: int (current iteration count)
- `mode`: str (PLANNING or EXECUTION)
- `memory`: AgentMemory (default factory instantiation)
- `plan`: str (planner's strategic plan text)
- `execution_instructions`: str (current instructions for executor)
- `needs_review`: bool (flag for forced review)

**Methods**:
- `increment_turn()`: Increments turn counter
- `update_screenshot(png)`: Updates screenshot buffer
- `get_context()`: Formats state for LLM consumption
  - Task description
  - Plan excerpt (400 chars with ellipsis)
  - Operating mode
  - Current turn number
  - Memory context

**Context Output Format**:
```
TASK: {task}

CURRENT_PLAN:
{plan_excerpt}

OPERATING_MODE: {mode}

CURRENT_TURN: {turn}

{memory_context}
```

### 9. LLM COMMUNICATION

**Function**: `post_json(payload)` (decorated with @log_api_call)

**Implementation**:
- JSON serialization with ASCII encoding
- urllib.request.Request with POST method
- Content-Type: application/json
- 240-second timeout
- Returns parsed JSON response

**Error Handling**: Relies on urllib exceptions (no explicit catch in function).

### 10. PLANNER AGENT

**System Prompt Structure**:

**Identity**: Computer Task Planner

**Responsibilities**:
- Step-by-step plan creation
- Progress review and plan adjustment
- Instruction delivery to executor
- NO direct action execution

**Tool Usage Rules**:
- Turn 1: Mandatory `update_execution_plan` for initial plan
- Strategy change: `update_execution_plan` with new approach
- Memory threshold: `archive_completed_actions` at 12+ active history items
- Review frequency: Every 7 turns

**Planning Guidelines**:
1. Decompose task into specific goals with success criteria
2. Each goal must be screenshot-verifiable
3. One instruction at a time to executor
4. Describe expected success indicators
5. BLOCKED status handling: provide alternative approach
6. DONE status handling: advance to next goal

**Function**: `invoke_planner(state)` → (Optional[str], Optional[ArchiveHistoryArgs])

**Turn 1 Behavior**:
- Prompt requests task analysis and initial plan creation
- Forces `update_execution_plan` tool usage

**Subsequent Turns**:
- Reviews recent executor actions
- Checks for compression need (12+ active history)
- Evaluates progress, failures, completions
- May update plan or compress memory

**Payload Construction**:
- Model: qwen3-vl-2b-instruct
- Temperature: 0.35 (lower than executor for consistency)
- Max tokens: 1800
- Tools: PLANNER_TOOLS (2 tools)
- Tool choice: auto

**Response Processing**:
- Extracts content for plan update
- Parses tool_calls array
- Returns instructions string if `update_execution_plan` called
- Returns archive arguments if `archive_completed_actions` called

### 11. EXECUTOR AGENT

**System Prompt Structure**:

**Identity**: Computer Task Executor

**Responsibilities**:
- Execute computer actions via tool calls
- Call exactly ONE tool per turn
- Validate previous action before next action

**Execution Protocol**:

**STEP_ONE: VALIDATE_PREVIOUS_ACTION**
- Examine current screenshot
- Check if previous action succeeded based on UI changes
- Document success/failure with visible proof

**STEP_TWO: CHOOSE_ONE_ACTION**
- Select one tool based on screenshot and planner instructions
- Provide reasoning in tool's reasoning field

**Required Response Structure**:
```
LAST_ACTION_RESULT: [validation statement]
CURRENT_GOAL: [specific goal being worked on]
SUCCESS_CRITERIA: [completion indicators]
SCREEN_SHOWS: [detailed screenshot description]
NEXT_STEP: [tool name and brief explanation]
```

**Coordinate System Documentation** (in prompt):
- 0-1000 grid for both axes
- Corner positions explicitly stated
- Side/position guidance (LEFT: 0-300, RIGHT: 700-1000, TOP: 0-300, BOTTOM: 700-1000)

**Critical Rules**:
1. ONE tool per turn enforcement
2. Previous action validation mandatory
3. Two consecutive failures → report goal BLOCKED
4. Goal completion → report_goal_status with DONE
5. Trust screenshot as ground truth
6. Precise coordinate positioning

**Function**: `invoke_executor(state)` → (Optional[ToolCall], str)

**Prerequisites Check**: Returns (None, "") if no execution_instructions exist

**Screenshot Handling**:
- Base64 encodes PNG from state
- Constructs multimodal message with text + image_url

**Previous Action Context Construction**:
- First turn: "No previous action"
- Subsequent turns: Detailed summary including:
  - Tool name
  - Action description with coordinates/text
  - Original reasoning
  - Expected result

**Payload Construction**:
- Model: qwen3-vl-2b-instruct
- Temperature: 0.5 (higher than planner for action diversity)
- Max tokens: 1400
- Tools: EXECUTOR_TOOLS (11 tools)
- Tool choice: auto
- Messages: system prompt + user prompt with image

**Response Processing**:
- Extracts content as response_text (structured validation text)
- Parses first tool_call if present
- Returns (ToolCall, response_text) or (None, response_text)

### 12. TOOL EXECUTION ENGINE

**Function**: `execute_tool(tool_call, sw, sh)` → str

**Click Tools** (click_screen_element, double_click_screen_element, right_click_screen_element):
- Validates ClickArgs type and element_name presence
- Converts normalized coordinates to physical pixels
- Executes mouse_move then appropriate click function
- Returns: "Clicked/Double-clicked/Right-clicked: {element_name}"

**Drag Tool** (drag_screen_element):
- Validates DragArgs type and element_name
- Converts start/end coordinates to pixels
- Executes mouse_drag with interpolation
- Returns: "Dragged: {element_name}"

**Type Tool** (type_text_input):
- Validates TypeTextArgs and text presence
- Executes keyboard_type_text
- Returns: "Typed: {text[:50]}"

**Press Tool** (press_keyboard_key):
- Validates PressKeyArgs and key presence
- Splits key combo on "+" delimiter
- Checks all parts exist in VK_MAP
- Executes keyboard_press_keys
- Returns: "Pressed key: {key}" or error

**Scroll Tools** (scroll_screen_down, scroll_screen_up):
- Moves mouse to screen center (sw//2, sh//2)
- Executes mouse_scroll with direction (-1 down, 1 up)
- Returns: "Scrolled down/up"

**Report Tool** (report_goal_status):
- Validates ReportProgressArgs
- Returns: "Goal status: {goal_identifier} -> {completion_status}"

**Error Handling**: Returns descriptive error strings for missing arguments or unknown tools.

### 13. LOOP RECOVERY MECHANISM

**Function**: `loop_recovery_action(state)`

**Trigger Conditions**:
- `enable_loop_recovery` config flag enabled
- More than 3 turns since last recovery (`loop_recovery_cooldown`)

**Recovery Actions** (sequential):
1. Press ESC key (cancel dialogs/menus)
2. Move mouse to screen center
3. Move mouse to bottom-left corner (Windows Start button region)
4. Left click
5. Press CTRL+ESC (open Start menu)

**State Update**: Marks current turn as last_recovery_turn

**Purpose**: Escape stuck UI states (modal dialogs, context menus, focus traps)

**Design Consideration**: Aggressive recovery may interfere with legitimate multi-step workflows. 3-turn cooldown prevents excessive interference.

### 14. MAIN AGENT LOOP

**Function**: `run_agent(state)` → str

**Loop Structure**: Iterates up to `max_steps` (50) turns

**Per-Iteration Logic**:

**Turn Increment**: Always first action

**Mode: PLANNING**
- Logs planning section
- Invokes `invoke_planner(state)`
- If archive_args returned: applies memory compression
- If instructions returned:
  - Updates state.execution_instructions
  - Switches mode to EXECUTION
  - Logs mode switch and instructions preview
- Sleeps turn_delay (1.5s)
- Continues to next iteration

**Mode: EXECUTION**

1. **Screenshot Capture**:
   - Captures PNG at configured resolution
   - Saves to screenshots directory
   - Updates state screenshot and dimensions

2. **Logging**:
   - Logs execution section header
   - Logs state update with turn and active history size

3. **Review Trigger Check**:
   - Turn divisible by review_interval (7)
   - OR needs_review flag set
   - OR memory.needs_compression (12+ active actions)
   - Action: Switch to PLANNING, clear needs_review flag, continue

4. **Executor Invocation**:
   - Calls `invoke_executor(state)`
   - If no tool_call returned: sleep and continue

5. **Completion Check**:
   - If tool_call.name == "report_mission_complete"
   - Validates evidence length >= 100 characters
   - If insufficient: log warning, sleep, continue
   - If sufficient: log completion section, return success message

6. **Tool Execution**:
   - Calls `execute_tool(tool_call, sw, sh)`
   - Logs tool execution with arguments and result

7. **Status Processing**:
   - If tool is report_goal_status with ReportProgressArgs:
     - Status "DONE"/"COMPLETED"/"COMPLETE": set needs_review flag
     - Status "BLOCKED": set needs_review flag

8. **Memory Recording**:
   - Creates ActionRecord with all context
   - Adds to state.memory

9. **Turn Delay**: Sleep 1.5s before next iteration

**Exit Conditions**:
- Mission completion reported: Returns "Mission completed in {turn} turns"
- Max steps reached: Returns "Max steps reached (50)"

### 15. MAIN ENTRY POINT

**Function**: `main()`

**Initialization Sequence**:

1. **DPI Setup**: Calls `init_dpi()` for coordinate accuracy

2. **Configuration Logging**:
   - Max steps
   - Review interval
   - Token limits for both agents

3. **Task Input**:
   - Prompts user for task description
   - Exits if empty string provided

4. **Initial Screenshot**:
   - Captures screen state before any actions
   - Saves as turn_0000.png
   - Provides baseline for planning

5. **State Creation**:
   - AgentState with task, initial screenshot, screen dimensions
   - Mode set to PLANNING for initial plan creation

6. **Agent Execution**:
   - Calls `run_agent(state)`
   - Receives final status string

7. **Debrief Logging**:
   - Final status
   - Total turns executed
   - Number of memory archives created

**Error Handling**: Exits with error message if no task provided. No exception handling for runtime errors (intentional fail-fast design).

## INTER-COMPONENT RELATIONSHIPS

### Agent Collaboration Flow
```
Planner Agent ──update_execution_plan──> AgentState.execution_instructions
                                              ↓
                                         Executor Agent
                                              ↓
                                         Tool Execution
                                              ↓
                                         ActionRecord
                                              ↓
                                         AgentMemory
                                              ↓
                            [Review Trigger: 7 turns / BLOCKED / DONE]
                                              ↓
                                         Planner Agent (loop)
```

### Memory Flow
```
ActionRecord ──add_action──> AgentMemory._history
                                    ↓
                          [12+ active records]
                                    ↓
                    Planner: archive_completed_actions
                                    ↓
                    apply_compression(summary, patterns, turns)
                                    ↓
                    Set archived=True on specified records
                                    ↓
                    Create MemorySnapshot
                                    ↓
                    get_context_for_llm() uses only active records
```

### Screenshot Pipeline
```
get_screen_size() ──> capture_png(1536, 864)
                            ↓
                    StretchBlt (downscale)
                            ↓
                    draw_cursor (overlay)
                            ↓
                    BGRA → RGB conversion
                            ↓
                    rgb_to_png (compression)
                            ↓
                    PNG bytes
                            ↓
                    ├──> LOGGER.save_screenshot (disk)
                    └──> AgentState.update_screenshot (memory)
                            ↓
                    base64 encoding
                            ↓
                    invoke_executor (LLM payload)
```

### Tool Call Pipeline
```
Executor LLM Response ──> tool_calls JSON array
                               ↓
                    ToolCall.parse(tool_call_dict)
                               ↓
                    Tool-specific argument parser lambda
                               ↓
                    Frozen dataclass (ClickArgs/TypeTextArgs/etc)
                               ↓
                    execute_tool(tool_call, sw, sh)
                               ↓
                    Windows API calls via ctypes
                               ↓
                    Result string
                               ↓
                    ActionRecord creation
                               ↓
                    Memory storage
```

## CRITICAL DESIGN DECISIONS

### 1. Stateless LLM Communication
**Decision**: Each API call includes full context via text formatting
**Rationale**: LM Studio /chat/completion endpoint has no session memory
**Implementation**: `get_context_for_llm()` reconstructs state from memory and plan
**Trade-off**: Higher token consumption vs. no server-side state management

### 2. Dual-Agent Architecture
**Decision**: Separate planner and executor roles
**Rationale**: Small model (4B parameters) benefits from role clarity
**Implementation**: Mode switching in AgentState, distinct tool sets, separate prompts
**Trade-off**: Increased turn count vs. reduced per-turn cognitive load

### 3. Coordinate Normalization
**Decision**: 0-1000 grid instead of physical pixels
**Rationale**: Resolution independence, simpler for small vision model
**Implementation**: Coordinate dataclass with to_pixels() conversion
**Trade-off**: Precision loss vs. model comprehension

### 4. Memory Compression
**Decision**: Archive old actions rather than delete
**Rationale**: Preserve historical patterns while reducing token count
**Implementation**: archived flag on ActionRecord, snapshot summaries
**Trade-off**: Complexity vs. context window management

### 5. Forced Review Intervals
**Decision**: Planner review every 7 turns regardless of progress
**Rationale**: Small model may not self-correct without external trigger
**Implementation**: Turn modulo check, mode switch to PLANNING
**Trade-off**: Planning overhead vs. loop detection

### 6. Single Tool Per Turn
**Decision**: Executor calls exactly one tool per turn
**Rationale**: Simplifies validation, reduces model confusion
**Implementation**: Prompt enforcement, parse first tool_call only
**Trade-off**: Higher turn count vs. model reliability

### 7. Screenshot-Based Validation
**Decision**: Executor must validate previous action from screenshot
**Rationale**: Stateless API requires visual confirmation of action success
**Implementation**: previous_action_info in prompt, structured response format
**Trade-off**: Turn overhead vs. error detection

### 8. PNG Compression Custom Implementation
**Decision**: Manual PNG encoding instead of PIL/Pillow
**Rationale**: Zero external dependencies beyond stdlib
**Implementation**: png_pack, rgb_to_png functions with zlib
**Trade-off**: Code complexity vs. dependency elimination

### 9. Hardcoded Timing Delays
**Decision**: Fixed delays (0.3s settle, 1.5s turn) instead of adaptive
**Rationale**: Simplicity, Windows 11 UI timing predictability
**Implementation**: time.sleep() calls after actions
**Trade-off**: Speed vs. reliability on target hardware

### 10. Loop Recovery Mechanism
**Decision**: Automatic ESC/click/Start menu sequence every 3+ turns
**Rationale**: Escape modal dialogs and stuck states without model understanding
**Implementation**: loop_recovery_action with cooldown tracking
**Trade-off**: Potential workflow disruption vs. stuck state recovery

## CONSTRAINT ANALYSIS

### Model Limitations (Qwen3-VL 4B)
1. **Small Parameter Count**: Requires unambiguous instructions, single tool per turn
2. **Vision Capabilities**: 1536x864 screenshot resolution necessary for element recognition
3. **Context Window**: Memory compression required beyond 12 actions
4. **Instruction Weighting**: Prompt warns against repetition causing importance bias
5. **Stateless API**: Full context reconstruction every turn

### Hardware Constraints
1. **125% DPI Scaling**: Requires DPI awareness context for coordinate accuracy
2. **1080p Display**: Screenshot downscaling from 1920x1080 to 1536x864
3. **Windows 11 UI Timing**: 0.3s settle delays calibrated for OS responsiveness
4. **LM Studio Local Inference**: 240s timeout accommodates local GPU inference time

### Design Constraints
1. **50 Turn Limit**: Hard stop prevents infinite loops
2. **7 Turn Review Interval**: Balances responsiveness vs. planning overhead
3. **3 Turn Recovery Cooldown**: Prevents recovery action interference
4. **100 Character Evidence Minimum**: Prevents premature completion reports
5. **No External Dependencies**: stdlib + ctypes only (no PIL, pyautogui, etc.)

## TOKEN BUDGET MANAGEMENT

### Planner Token Budget (1800)
- System prompt: ~600 tokens
- Context (task + plan excerpt + memory): ~800 tokens
- User prompt: ~200 tokens
- Response buffer: ~200 tokens

### Executor Token Budget (1400)
- System prompt: ~700 tokens
- Context (task + instructions + memory): ~400 tokens
- Screenshot (base64 overhead): Token cost handled by LM Studio
- User prompt: ~150 tokens
- Response buffer: ~150 tokens

### Memory Compression Triggers
- Active history: 12 actions = ~300-400 tokens
- Compressed snapshot: ~50 tokens per archive
- Net savings: ~250 tokens per compression cycle

## FAILURE MODES AND MITIGATIONS

### 1. Model Hallucination
**Symptom**: Tool calls with invalid coordinates or arguments
**Mitigation**: Coordinate clamping in to_pixels(), VK_MAP validation in press_keyboard_key

### 2. Action Validation Failure
**Symptom**: Model doesn't recognize action success/failure
**Mitigation**: Structured response format with LAST_ACTION_RESULT section, screenshot-based validation

### 3. Stuck UI States
**Symptom**: Modal dialogs, context menus blocking progress
**Mitigation**: loop_recovery_action with ESC key and Start menu sequence

### 4. Infinite Loops
**Symptom**: Repeated identical actions
**Mitigation**: Review interval triggering, memory pattern detection, max_steps limit

### 5. Premature Completion
**Symptom**: Mission complete without full task execution
**Mitigation**: 100-character evidence minimum, planner review of completion claims

### 6. Context Window Overflow
**Symptom**: API errors from excessive history
**Mitigation**: Memory compression at 12 action threshold, snapshot summarization

### 7. Coordinate Misalignment
**Symptom**: Clicks miss intended targets
**Mitigation**: DPI awareness initialization, explicit corner/side coordinate guidance in prompt

### 8. Timing Races
**Symptom**: Actions execute before UI ready
**Mitigation**: ui_settle_delay (0.3s) after every action, turn_delay (1.5s) between turns

## SECURITY CONSIDERATIONS

### Input Validation
- Task string: No validation (accepts arbitrary Unicode)
- Tool arguments: Type checking in parsers, coordinate clamping
- Key combinations: VK_MAP whitelist prevents arbitrary virtual key codes

### Execution Privileges
- Runs with user's Windows privileges
- No elevation mechanism
- SendInput bypasses UIPI (can't interact with elevated processes)

### Data Exposure
- Screenshots saved unencrypted to dumps/ directory
- API logs contain full conversation history
- No credential handling or sanitization

### Uncontrolled Execution
- No sandboxing of tool execution
- Can interact with any accessible UI
- Loop recovery may close unrelated dialogs

## PERFORMANCE CHARACTERISTICS

### Per-Turn Latency
- Screenshot capture: ~50-100ms
- API call (LM Studio local): ~5-15s (model dependent)
- Tool execution: ~50-500ms (action dependent)
- Delays: 1.5s turn_delay
- **Total**: ~7-17s per turn typical

### Memory Footprint
- Screenshot buffer: ~1-2 MB (1536x864 PNG)
- Action history: ~100 KB per 12 actions
- Snapshots: ~1 KB per compression
- **Estimated**: ~5-10 MB after 50 turns

### API Call Frequency
- Planner: Turn 1 + every 7 turns + on-demand reviews
- Executor: Every execution turn
- **Total**: ~45 calls typical for 50-turn run (5 planner, 40 executor)

### Throughput
- 50 turns maximum
- ~7-17s per turn
- **Total runtime**: 6-15 minutes for full run

## EXTENSIBILITY POINTS

### Adding New Tools
1. Define frozen dataclass for arguments in tool schemas section
2. Add tool definition dict to PLANNER_TOOLS or EXECUTOR_TOOLS
3. Add parser lambda to ToolCall.parse() parsers dict
4. Add execution case to execute_tool() function
5. Update agent prompt with tool description and usage rules

### Modifying Prompts
- PLANNER_PROMPT and EXECUTOR_PROMPT are module-level string constants
- Use .format() for variable injection (task, instructions, etc.)
- Maintain structured sections for model role clarity

### Adjusting Configuration
- Modify Config dataclass defaults
- All components reference CFG singleton
- No runtime configuration mechanism (recompilation required)

### Custom Memory Strategies
- Subclass AgentMemory
- Override get_context_for_llm() and apply_compression()
- Inject custom instance into AgentState initialization

### Alternative Input/Output
- Replace mouse/keyboard functions with remote desktop protocol
- Replace capture_png with VNC/RDP screenshot
- Maintain same interface signatures

## DEPLOYMENT REQUIREMENTS

### Mandatory Components
1. Windows 11 Pro 25H2 (user32.dll, gdi32.dll APIs)
2. Python 3.12.10 (ctypes, dataclasses, typing features)
3. LM Studio 0.3.39 with qwen3-vl-4b-instruct loaded
4. OpenAI-compatible endpoint at localhost:1234

### File System
- Write permissions to dumps/ directory (current working directory)
- screenshots/ subdirectory auto-created

### Network
- HTTP access to localhost:1234
- No internet connectivity required

### Display
- Active display output (cannot run headless)
- 1920x1080 resolution recommended (125% scaling configured)
- Cursor visible for overlay capture

### Permissions
- No administrator privileges required
- User-level input simulation (SendInput)
- Desktop interaction permissions (GetDC)

## OPERATIONAL NOTES

### First Run Setup
1. Create dumps/ directory or allow auto-creation
2. Start LM Studio with model loaded
3. Verify endpoint responds at http://localhost:1234/v1/chat/completions
4. Launch script and provide task description

### Task Specification Best Practices
- Use concrete, verifiable objectives
- Specify exact application names and file paths
- Describe success state clearly
- Avoid ambiguous terms (e.g., "find the button" vs. "click the Save button")

### Monitoring Execution
- Watch execution_log.txt for real-time progress
- Screenshots numbered sequentially for debugging
- API call numbers correlate requests with responses

### Interruption Handling
- CTRL+C will terminate (no graceful shutdown)
- Partial execution logs preserved
- No resume mechanism (restart from beginning)

### Debugging Failed Runs
1. Check screenshots/ for visual execution trace
2. Review execution_log.txt for tool calls and results
3. Examine API requests/responses for prompt issues
4. Check last screenshot against expected UI state

---

**END OF TECHNICAL ANALYSIS REPORT**
