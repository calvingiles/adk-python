# ADK Approval Plugin: Refactoring Plan

## Executive Summary

This document outlines the plan to refactor the substantial approval mechanism work from the `feature/approval-mechanism` branch into a standalone plugin that aligns with ADK's current architecture. The original work introduced a comprehensive human-in-the-loop approval system with ~6,000 lines of code changes, including core approval logic, policies, grants, and extensive test coverage.

**Key Decision**: Refactor the approval system as an **external plugin package** (`adk-approvals-plugin`) rather than a core ADK change, aligning with the project's plugin architecture direction.

---

## Analysis of Original Work

### Original Implementation (feature/approval-mechanism branch)

#### Core Components Created
1. **`src/google/adk/approval/`** - Complete approval module (5 files, ~1,300 LOC)
   - `approval_policy.py` - Policy definitions and registry
   - `approval_handler.py` - Core approval logic (~500 LOC)
   - `approval_grant.py` - Grant/actor data structures
   - `approval_request.py` - Request/response models
   - `approval_request_processor.py` - LLM request processor integration (~300 LOC)

2. **Integration Points**
   - Modified `flows/llm_flows/functions.py` - Injected approval checking before tool execution
   - Modified `agents/llm_agent.py` - Minor cleanup
   - Modified `tools/tool_context.py` - Added `request_approval()` method
   - Modified `events/event_actions.py` - Added `requested_approvals` field
   - Integration with session state for grants/suspended calls

3. **Test Coverage**
   - ~1,200 LOC of comprehensive tests
   - Tests for policies, grants, handlers, and end-to-end flows
   - Mock tools with `@tool_policy` decorator

4. **Plugin Infrastructure**
   - Introduced `BasePlugin` class (commit 4dce9ef) - 320 LOC
   - Lifecycle callbacks: before/after agent, tool, model, run
   - Plugin execution order and short-circuiting logic

#### Key Features
- **Policy-Based Control**: Declarative `@tool_policy` decorator
- **Grant Management**: Allow/deny grants with expiration
- **Hierarchical Actors**: User → Agent → Tool delegation chain
- **State Management**: Session-based grant and suspended call tracking
- **Challenge Flow**: Multi-round approval with partial grants
- **Action-Resource Model**: Fine-grained permission (action × resource)

#### Architecture Pattern
The original implementation uses an **LLM Request Processor** pattern:
- Intercepts function calls before execution
- Checks policies against grants
- Suspends calls pending approval
- Resumes calls when grants received
- Uses special function call `adk_request_approval` for UI communication

---

## Current ADK Architecture Analysis

### Existing Extension Mechanisms (Main Branch)

ADK provides several extension points, but **no formal plugin system yet** (the `BasePlugin` was on the approval branch):

1. **Agent Callbacks** - Per-agent hooks
   - `before_agent_callback`, `after_agent_callback`
   - `before_tool_callback`, `after_tool_callback`
   - `before_model_callback`, `after_model_callback`

2. **LLM Request/Response Processors** - Flow-level middleware
   - `BaseLlmRequestProcessor` - Modify requests before LLM
   - `BaseLlmResponseProcessor` - Process responses after LLM
   - Examples: `_CodeExecutionProcessor`, `_NlPlanningProcessor`

3. **Tool System** - Extensible tool interface
   - `BaseTool` - Base class for all tools
   - `FunctionTool` - Wraps Python functions
   - `LongRunningFunctionTool` - Async approval pattern (see `human_in_loop` sample)

4. **Auth System** - Pluggable auth credential exchangers
   - `BaseAuthCredentialExchanger` - Auth plugin interface
   - `AutoAuthCredentialExchanger` - Registry with custom exchangers

### Existing Human-in-the-Loop Pattern

ADK's `human_in_loop` sample uses **Long-Running Tools**:
- Tool returns `{status: 'pending', ticketId: '...'}` immediately
- External system processes approval
- Client sends `FunctionResponse` with updated status
- Agent continues based on response

**Limitations of this approach**:
- No policy framework
- Manual approval logic in each tool
- No grant persistence/reuse
- No declarative policy configuration

---

## Comparison with Industry Standards

### LangChain/LangGraph
- **LangChain**: `HumanApprovalCallbackHandler` - callback-based, per-tool configuration
- **LangGraph**: Checkpoint-based state persistence with graph pause/resume

### Amazon Bedrock Agents
- **User Confirmation**: Boolean approve/reject
- **Return of Control (ROC)**: Form-based intent editing

### OpenAI Agents SDK
- Human-in-the-loop via tool confirmation callbacks

### Our Approach Advantages
✅ **Policy-based**: Declarative policies vs. imperative callbacks
✅ **Grant persistence**: Reusable grants across calls
✅ **Fine-grained**: Action-resource model (vs. boolean)
✅ **Hierarchical**: Actor delegation chain
✅ **Challenge flow**: Partial approval support

---

## Proposed Plugin Architecture

### Design Philosophy

**Plugin as External Package**: Create `adk-approvals-plugin` as a separate installable package that extends ADK via the plugin architecture.

#### Why Plugin vs. Core?
1. ✅ **Modularity**: Users opt-in to approval functionality
2. ✅ **Faster Iteration**: Independent release cycle from ADK core
3. ✅ **Reduced Complexity**: Core ADK remains focused
4. ✅ **Easier Contribution**: Clear plugin contribution path
5. ✅ **Demonstrates Plugin Pattern**: Reference implementation for community

### Architecture Layers

```
┌─────────────────────────────────────────────────┐
│  adk-approvals-plugin (External Package)        │
├─────────────────────────────────────────────────┤
│  1. ApprovalPlugin (BasePlugin implementation)  │
│     - before_tool_callback: Check policies      │
│     - on_event_callback: Handle grant updates   │
│  2. Policy Framework                            │
│     - @tool_policy decorator                    │
│     - ApprovalPolicyRegistry                    │
│  3. Grant Management                            │
│     - ApprovalGrant, ApprovalActor              │
│     - Grant storage in session state            │
│  4. UI Integration                              │
│     - Special function call: request_approval   │
│     - Challenge/response flow                   │
├─────────────────────────────────────────────────┤
│  ADK Core (Minimal Changes)                     │
├─────────────────────────────────────────────────┤
│  1. Plugin System (from approval branch)        │
│     - BasePlugin abstract class                 │
│     - Plugin registration in Runner             │
│     - Callback execution order                  │
│  2. Event Actions Extension (already exists)    │
│     - requested_approvals field                 │
│  3. ToolContext Extension (minimal)             │
│     - request_approval() helper method          │
└─────────────────────────────────────────────────┘
```

### Plugin Integration Points

#### 1. Tool Execution Interception
```python
class ApprovalPlugin(BasePlugin):
    async def before_tool_callback(
        self, *, tool: BaseTool, tool_args: dict, tool_context: ToolContext
    ) -> Optional[dict]:
        # Check if tool has policies
        policies = ApprovalPolicyRegistry.get_tool_policies(tool.name)
        if not policies:
            return None  # No approval needed

        # Check existing grants
        grants = self._get_grants(tool_context.state)
        if self._is_approved(policies, tool_args, grants):
            return None  # Already approved, proceed

        # Request approval
        approval_request = self._create_approval_request(policies, tool_args)
        tool_context.request_approval(approval_request)

        # Suspend this call
        self._suspend_function_call(tool_context, tool.name, tool_args)
        return {"status": "approval_requested"}
```

#### 2. Grant Update Handling
```python
class ApprovalPlugin(BasePlugin):
    async def on_event_callback(
        self, *, invocation_context: InvocationContext, event: Event
    ) -> Optional[Event]:
        # Check for approval responses
        approval_responses = self._extract_approval_responses(event)
        if not approval_responses:
            return None

        # Update grants in state
        new_grants = self._process_approval_responses(
            approval_responses,
            invocation_context.session.state
        )

        # Resume suspended calls with new grants
        resumed_calls = self._resume_suspended_calls(
            invocation_context.session.state,
            new_grants
        )

        # Generate events for resumed calls
        return self._create_resumed_calls_event(resumed_calls)
```

#### 3. Policy Registration
```python
# In user code
from adk_approvals_plugin import tool_policy, ApprovalAction

@tool_policy(
    actions=[ApprovalAction("bigquery:query:write")],
    resources=lambda args: [f"bigquery:table:{args['dataset']}.{args['table']}"]
)
def insert_bigquery_data(dataset: str, table: str, data: dict) -> dict:
    # Implementation
    pass
```

---

## Implementation Plan

### Phase 1: Extract Plugin Infrastructure to Core ADK

**Goal**: Bring `BasePlugin` from approval branch to main, enabling plugin ecosystem.

**Tasks**:
1. Cherry-pick commit 4dce9ef (BasePlugin) to new branch from main
2. Update `Runner` to accept `plugins: list[BasePlugin]` parameter
3. Implement plugin callback execution order:
   - Plugins execute before agent callbacks
   - First non-None return short-circuits
4. Add plugin callback invocation to appropriate lifecycle points:
   - In `Runner.run_async()`: `before_run_callback`, `after_run_callback`, `on_event_callback`
   - In `BaseAgent`: `before_agent_callback`, `after_agent_callback`
   - In function handling: `before_tool_callback`, `after_tool_callback`
   - In LLM calls: `before_model_callback`, `after_model_callback`
5. Add `on_user_message_callback` support
6. Write unit tests for plugin system
7. Create plugin documentation and sample (simple logging plugin)

**Deliverable**: PR to ADK main with plugin infrastructure

**Files Changed**:
- `src/google/adk/plugins/__init__.py` (new)
- `src/google/adk/plugins/base_plugin.py` (new)
- `src/google/adk/runners.py` (modified - add plugin support)
- `src/google/adk/agents/base_agent.py` (modified - plugin callback integration)
- `src/google/adk/flows/llm_flows/functions.py` (modified - tool callback integration)
- `tests/unittests/plugins/test_base_plugin.py` (new)
- `docs/plugins/creating_plugins.md` (new)

**Estimated Effort**: 3-5 days

---

### Phase 2: Create Approval Plugin Package Skeleton

**Goal**: Set up external `adk-approvals-plugin` package structure.

**Tasks**:
1. Create new repository: `adk-approvals-plugin`
2. Set up Python package structure:
   ```
   adk-approvals-plugin/
   ├── pyproject.toml
   ├── README.md
   ├── LICENSE (Apache 2.0)
   ├── src/
   │   └── adk_approvals_plugin/
   │       ├── __init__.py
   │       ├── plugin.py          # ApprovalPlugin class
   │       ├── policy.py          # Policies and registry
   │       ├── grant.py           # Grants and actors
   │       ├── request.py         # Request/response models
   │       └── handler.py         # Core approval logic
   ├── tests/
   │   └── test_approval_plugin.py
   └── examples/
       └── bigquery_with_approvals/
   ```
3. Add dependency: `google-adk >= 0.3.0` (or version with plugin support)
4. Set up CI/CD (GitHub Actions)
5. Configure linting (pylint/black matching ADK)

**Deliverable**: Empty package ready for code migration

**Estimated Effort**: 1-2 days

---

### Phase 3: Migrate Core Approval Logic to Plugin

**Goal**: Move approval mechanism from ADK core to plugin package.

**Tasks**:
1. **Copy and adapt core modules**:
   - `approval_grant.py` → `src/adk_approvals_plugin/grant.py`
   - `approval_policy.py` → `src/adk_approvals_plugin/policy.py`
   - `approval_request.py` → `src/adk_approvals_plugin/request.py`
   - `approval_handler.py` → `src/adk_approvals_plugin/handler.py`

2. **Create ApprovalPlugin class** (`plugin.py`):
   ```python
   from google.adk.plugins import BasePlugin

   class ApprovalPlugin(BasePlugin):
       def __init__(self, name: str = "approval_plugin"):
           super().__init__(name)

       async def before_tool_callback(self, *, tool, tool_args, tool_context):
           # Port logic from approval_handler.get_approval_request()
           pass

       async def on_event_callback(self, *, invocation_context, event):
           # Port logic from _ApprovalLlmRequestProcessor.run_async()
           pass
   ```

3. **Update imports and namespaces**:
   - Change from `google.adk.approval.*` to `adk_approvals_plugin.*`
   - Update all internal references

4. **Adapt state management**:
   - Continue using session state: `state["approvals__grants"]`
   - Continue using: `state["approvals__suspended_function_calls"]`
   - Document state schema for users

5. **Remove dependencies on internal ADK changes**:
   - Approval logic should work via public plugin API only
   - If `tool_context.request_approval()` doesn't exist, provide workaround

**Deliverable**: Functional approval plugin (not yet tested)

**Estimated Effort**: 5-7 days

---

### Phase 4: Implement UI Integration

**Goal**: Handle approval request/response flow with ADK Runner.

**Tasks**:
1. **Define approval function call protocol**:
   ```python
   REQUEST_APPROVAL_FUNCTION_NAME = "adk_request_approval"

   # Plugin emits function call:
   types.FunctionCall(
       name=REQUEST_APPROVAL_FUNCTION_NAME,
       args=ApprovalRequest(...).model_dump(),
   )

   # Client responds with:
   types.FunctionResponse(
       name=REQUEST_APPROVAL_FUNCTION_NAME,
       response=ApprovalResponse(grants=[...]).model_dump(),
   )
   ```

2. **Use `tool_context.request_approval()` or emit events**:
   - Check if ADK core has `request_approval()` method
   - If not, emit approval request via `Event.actions.requested_approvals`
   - Document both approaches

3. **Create UI helper utilities**:
   ```python
   # In plugin package
   def format_approval_request_for_ui(approval_request: ApprovalRequest) -> dict:
       """Convert approval request to user-friendly format."""
       pass

   def create_grant_from_user_input(user_response: dict) -> ApprovalGrant:
       """Create grant from UI response."""
       pass
   ```

4. **Example UI integration** (for docs):
   - Show how to handle `adk_request_approval` in client
   - Show how to construct `ApprovalResponse`
   - Provide React/CLI UI sample code

**Deliverable**: Complete approval request/response cycle

**Estimated Effort**: 3-4 days

---

### Phase 5: Testing and Validation

**Goal**: Comprehensive test coverage for plugin.

**Tasks**:
1. **Port existing tests** from `tests/unittests/approval/`:
   - `test_approval_grant.py`
   - `test_approval_policy.py`
   - `test_approval_handler.py`
   - `test_approval_preprocessor.py` → adapt for plugin callbacks
   - `test_approval.py` (integration tests)

2. **Update tests for plugin architecture**:
   - Replace LLM request processor assertions with plugin callback assertions
   - Test plugin registration with Runner
   - Test callback execution order
   - Test short-circuiting behavior

3. **Add new tests**:
   - Test plugin with multiple agents
   - Test plugin + agent callbacks interaction
   - Test plugin with different tool types (function, long-running, etc.)
   - Test error handling and edge cases

4. **Integration tests with real ADK**:
   - Test with actual Runner and agents
   - Test with BigQuery tools (original use case)
   - Test with custom tools

5. **Performance testing**:
   - Measure overhead of approval checking
   - Optimize policy lookup if needed

**Deliverable**: >90% test coverage, all tests passing

**Estimated Effort**: 5-7 days

---

### Phase 6: Documentation and Examples

**Goal**: Comprehensive documentation for plugin users.

**Tasks**:
1. **README.md** with:
   - Quick start guide
   - Installation instructions
   - Basic usage example
   - Link to full documentation

2. **Full documentation** (in `docs/` or GitHub Wiki):
   - **Getting Started**:
     - Installation
     - Basic approval flow
     - First policy
   - **Core Concepts**:
     - Approval policies
     - Grants and actors
     - Action-resource model
     - Challenge flow
   - **API Reference**:
     - `ApprovalPlugin` configuration
     - Policy decorators and functions
     - Grant creation and management
     - Request/response formats
   - **Advanced Topics**:
     - Custom resource mappers
     - Grant expiration
     - Deny policies
     - Multi-agent approvals
   - **UI Integration**:
     - Handling approval requests
     - Creating approval UI
     - Example implementations (CLI, web)

3. **Example applications**:
   - **BigQuery with approvals**: Require approval for write queries
   - **File system agent**: Approve file deletions
   - **Multi-step workflow**: Approvals at critical steps
   - **Budget control**: Approve expensive API calls

4. **Migration guide** (for your original work):
   - How original core integration maps to plugin
   - Changes needed to adopt plugin version

**Deliverable**: Complete documentation and working examples

**Estimated Effort**: 4-6 days

---

### Phase 7: Polish and Release

**Goal**: Production-ready plugin package.

**Tasks**:
1. **Code quality**:
   - Run linters (pylint, mypy)
   - Address all warnings
   - Add type hints everywhere
   - Add docstrings for all public APIs

2. **Security review**:
   - Review grant validation logic
   - Check for privilege escalation risks
   - Validate input sanitization
   - Review actor chain integrity

3. **Package publishing**:
   - Configure PyPI publishing
   - Create version 0.1.0
   - Generate changelog
   - Tag release in Git

4. **Community engagement**:
   - Create announcement post
   - Submit to ADK samples/plugins directory
   - Create demo video
   - Engage on Reddit/Discord

**Deliverable**: Published `adk-approvals-plugin` v0.1.0

**Estimated Effort**: 3-4 days

---

## Detailed Technical Considerations

### 1. Plugin Callback Mapping

| Original Approach | Plugin Approach |
|-------------------|-----------------|
| `_ApprovalLlmRequestProcessor.run_async()` | `ApprovalPlugin.on_event_callback()` - process approval responses |
| `ApprovalHandler.get_approval_request()` in function handler | `ApprovalPlugin.before_tool_callback()` - check policies |
| `@tool_policy` decorator | Same - part of plugin package |
| State management in session | Same - use session.state |
| `REQUEST_APPROVAL_FUNCTION_CALL_NAME` | Same - special function call |

### 2. State Schema

The plugin will use the following state keys (same as original):

```python
state = {
    "approvals__grants": [
        {
            "effect": "allow",
            "actions": ["tool:bigquery:query"],
            "resources": ["bigquery:project.dataset.table"],
            "grantee": {...},
            "grantor": {...},
            "expiration_time": "2025-10-22T00:00:00Z",
            "comment": "Approved for analysis"
        }
    ],
    "approvals__suspended_function_calls": [
        {
            "function_call": {"id": "abc", "name": "query", "args": {...}},
            "status": "suspended",
            "sequence": 1
        }
    ]
}
```

### 3. Core ADK Changes Needed

**Minimal changes to ADK core** (only plugin infrastructure):

1. **Add `BasePlugin` class** (already designed in approval branch)
2. **Add plugin support to `Runner`**:
   ```python
   class Runner:
       def __init__(self, agent: BaseAgent, plugins: list[BasePlugin] = None):
           self.plugins = plugins or []
   ```
3. **Add plugin callback invocation** at appropriate lifecycle points
4. **Add `requested_approvals` to `EventActions`** (already exists in approval branch)
5. **Optional: Add `tool_context.request_approval()` helper** (can be workaround in plugin)

**No changes needed** to:
- Tool system
- Flow system (no custom processors)
- Agent system (uses standard callbacks)
- Session system (uses standard state)

### 4. Dependency Management

```toml
# pyproject.toml for adk-approvals-plugin
[project]
name = "adk-approvals-plugin"
version = "0.1.0"
dependencies = [
    "google-adk>=0.3.0",  # Requires plugin support
    "pydantic>=2.0.0",
]
```

### 5. Backward Compatibility

**For your original work**:
- Original branch used core integration → now uses plugin
- Migration path: Install plugin, register with Runner
- State schema unchanged → grants/suspended calls compatible
- Policy decorator unchanged → API compatible

**For new users**:
- Clean plugin installation
- No breaking changes to core ADK
- Opt-in via plugin registration

---

## Migration Path (From Original Branch)

For anyone using the original `feature/approval-mechanism` branch:

### Before (Core Integration)
```python
from google.adk import Agent
from google.adk.approval import tool_policy, ApprovalAction

@tool_policy(
    actions=[ApprovalAction("write")],
    resources=lambda args: ["bigquery:table"]
)
def write_data(data): ...

agent = Agent(tools=[write_data], ...)
# Approval processor automatically added to flow
```

### After (Plugin)
```python
from google.adk import Agent, Runner
from adk_approvals_plugin import ApprovalPlugin, tool_policy, ApprovalAction

@tool_policy(
    actions=[ApprovalAction("write")],
    resources=lambda args: ["bigquery:table"]
)
def write_data(data): ...

agent = Agent(tools=[write_data], ...)
runner = Runner(
    agent=agent,
    plugins=[ApprovalPlugin()]  # ← Only change: register plugin
)
```

**Changes Required**:
1. Install: `pip install adk-approvals-plugin`
2. Change imports: `google.adk.approval` → `adk_approvals_plugin`
3. Register plugin with Runner
4. Everything else identical (state, policies, grants, UI)

---

## Testing Strategy

### Unit Tests
- Policy registration and matching
- Grant validation logic
- Actor hierarchy
- Resource mapping
- Request/response serialization

### Integration Tests
- Plugin + Runner integration
- Plugin callback execution order
- State persistence across calls
- Multi-round approval flow
- Error handling

### End-to-End Tests
- Complete approval workflow with real agent
- BigQuery approval scenario
- File system approval scenario
- Multi-agent coordination

### Performance Tests
- Policy lookup overhead
- Grant checking performance
- State serialization cost
- Memory usage with many grants

---

## Documentation Plan

### User Documentation
1. **Installation Guide**
2. **Quick Start Tutorial** (5-minute approval setup)
3. **Core Concepts** (policies, grants, actors)
4. **API Reference** (auto-generated from docstrings)
5. **Example Gallery** (BigQuery, files, budgets)
6. **UI Integration Guide** (building approval UIs)
7. **Troubleshooting** (common issues)

### Developer Documentation
1. **Architecture Overview** (how plugin works internally)
2. **Contributing Guide** (extending the plugin)
3. **Testing Guide** (running and writing tests)
4. **Release Process**

### Video Content
1. **Demo Video**: "Adding Approval to Your ADK Agent" (3-5 min)
2. **Deep Dive**: "Understanding the Approval Flow" (10-15 min)

---

## Timeline Estimate

| Phase | Tasks | Duration |
|-------|-------|----------|
| 1. Plugin Infrastructure to Core | BasePlugin + Runner integration | 3-5 days |
| 2. Package Skeleton | Repo setup, structure | 1-2 days |
| 3. Migrate Core Logic | Port approval code to plugin | 5-7 days |
| 4. UI Integration | Approval request/response flow | 3-4 days |
| 5. Testing | Comprehensive test suite | 5-7 days |
| 6. Documentation | Docs + examples | 4-6 days |
| 7. Polish & Release | QA, publish | 3-4 days |
| **Total** | | **24-35 days** |

**Realistic Timeline**: 5-7 weeks (accounting for reviews, iterations)

---

## Success Criteria

### Must Have
✅ Plugin installs cleanly via pip
✅ Works with ADK 0.3.0+ (with plugin support)
✅ All original approval functionality preserved
✅ >90% test coverage
✅ Complete documentation with examples
✅ Zero changes to core ADK tool/agent/flow systems

### Should Have
✅ Performance overhead <5% for non-approval cases
✅ Published to PyPI
✅ Example applications (BigQuery, files)
✅ Migration guide from original branch

### Nice to Have
✅ UI component library (React, Vue)
✅ Integration with ADK Web UI
✅ Admin dashboard for grant management
✅ Audit logging plugin integration

---

## Risks and Mitigations

### Risk: Plugin API Insufficient
**Mitigation**: Phase 1 validates plugin API with simple plugin first. If gaps found, extend `BasePlugin` before Phase 3.

### Risk: Performance Overhead
**Mitigation**: Performance tests in Phase 5. Optimize policy registry with caching/indexing if needed.

### Risk: State Serialization Issues
**Mitigation**: Keep state schema simple (JSON-serializable). Add migration path for schema changes.

### Risk: UI Integration Complexity
**Mitigation**: Provide multiple UI examples (CLI, web). Create helper utilities to simplify integration.

### Risk: Community Adoption
**Mitigation**: Comprehensive docs, video demos, showcase in ADK samples. Engage early with potential users for feedback.

---

## Next Steps

### Immediate Actions
1. **Review and approve this plan** with stakeholders
2. **Create Phase 1 branch** from current main
3. **Set up project tracking** (GitHub Project/Issues)
4. **Assign initial tasks** for Phase 1

### Discussion Points
- Should plugin infrastructure (Phase 1) be a separate PR or combined with approval plugin?
- Naming: `adk-approvals-plugin` vs `adk-approval-plugin` vs other?
- Repository location: New repo or ADK monorepo subfolder?
- Release strategy: Alpha/beta releases or direct to 0.1.0?

### Open Questions
1. Does ADK core team want to maintain plugin as official extension?
2. Should we create `adk-plugins` organization for community plugins?
3. Are there other approval/auth use cases we should consider?
4. Should we integrate with existing IAM systems (Google IAM, AWS IAM)?

---

## Conclusion

This plan transforms your substantial approval mechanism work into a clean, reusable plugin that demonstrates ADK's extensibility while preserving all the sophisticated functionality you built. The plugin approach:

- **Preserves your work**: All 6,000+ LOC of functionality maintained
- **Aligns with ADK vision**: Demonstrates plugin architecture
- **Enables contribution**: Clear path to contribute as external package
- **Provides flexibility**: Users opt-in to approvals when needed
- **Ensures quality**: Comprehensive testing and documentation

The refactoring is substantial but well-scoped, with clear phases and deliverables. By the end, you'll have a production-ready plugin that showcases sophisticated human-in-the-loop approval with ADK's blessing as a reference plugin implementation.

**Ready to proceed?** Let's discuss any questions or adjustments to the plan, then kick off Phase 1!
