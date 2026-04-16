[commit_aware_testing_agent_prd.md](https://github.com/user-attachments/files/26769120/commit_aware_testing_agent_prd.md)
# PRD: Commit-Aware Impact Analysis and Test Generation Agent

## Document Control
- **Version:** 1.0
- **Status:** Draft
- **Owner:** Engineering / AI Platform
- **Primary Audience:** Coding agent implementing the testing agent pipeline
- **Last Updated:** 2026-04-16

## 1. Summary
Build a commit-aware testing agent for an actively changing monorepo with a Next.js frontend and FastAPI backend. Given a prompt such as `test change in commit 33cffa`, the agent must determine what changed, identify the affected code paths across frontend and backend, extract real route and model parameters from source-of-truth code artifacts, and generate grounded test plans and test cases.

The system must prevent premature test generation. The agent is not allowed to suggest tests until it has completed a structured impact-analysis workflow and passed explicit completeness checks.

## 2. Problem Statement
The current agent often proposes tests before it has completed enough research. In practice this causes:
- Missed affected routes and components
- Missing or invented request parameters
- Incomplete coverage of downstream dependencies
- Weak handling of route variants, auth requirements, and edge cases
- Over-reliance on embeddings or shallow retrieval rather than source-grounded analysis

The desired system must behave more like a deterministic analyst than a free-form chatbot.

## 3. Goals
### Primary Goals
- Accurately map a commit or diff to affected backend routes, frontend components, and shared data models
- Extract route parameters, request bodies, response schemas, auth requirements, and relevant edge conditions from code, not from model guesses
- Generate a structured impact report before generating tests
- Produce test cases grounded in the impact report and source-of-truth schema extraction
- Stay fresh in an actively changing monorepo via incremental ingestion and on-demand file reads

### Secondary Goals
- Explain why each suggested test exists
- Support both route-level and component-level impact analysis
- Make the analysis traceable and inspectable for developers

## 4. Non-Goals
- Full formal program analysis or complete call-graph precision
- Rebuilding the entire code index on every request
- Fully autonomous code modification or PR creation
- Replacing existing unit/integration/E2E frameworks
- One-shot test generation from embeddings alone

## 5. Users and Key Use Cases
### Primary User
- An engineer or QA-focused AI workflow that asks the agent to analyze a commit and propose tests

### Example Prompts
- `test change in commit 33cffa`
- `analyze impact of HEAD~2..HEAD and propose regression tests`
- `what tests should run for this PR?`
- `did this change affect auth, filtering, or pagination behavior?`

## 6. Product Requirements
### 6.1 Inputs
The system must support at least the following inputs:
- A single commit hash
- A commit range
- A working tree diff or staged diff
- Optional scope constraints such as `backend only`, `frontend only`, or `critical paths only`

### 6.2 Outputs
The system must produce two distinct artifacts:

#### A. Impact Analysis Report
A structured report containing:
- Changed files
- Changed symbols if determinable
- Affected backend routes
- Affected frontend routes/pages/components
- Affected request and response models
- Dependency paths linking changes to impacted behaviors
- Extracted parameters and schemas for each impacted route
- Confidence and unresolved gaps

#### B. Test Plan / Test Cases
A grounded set of proposed tests categorized by:
- Happy path
- Validation and bad input
- Authorization / authentication
- Edge cases / boundary cases
- Regression risks
- Frontend rendering / state handling
- Integration paths across frontend and backend

The agent must never produce artifact B without artifact A being complete enough.

## 7. Core Design Principle
**Do not try to make the model smarter through prompt wording alone. Make the workflow stricter.**

This system must enforce a staged, tool-based analysis process with validation gates.

## 8. Functional Requirements
### 8.1 Mandatory Multi-Phase Workflow
The agent must follow this sequence:
1. Resolve diff scope
2. Extract changed files and changed symbols
3. Map changed files to routes, models, components, and API clients
4. Expand impact through dependency traversal
5. Extract route schemas and parameter details from source-of-truth code
6. Validate completeness of analysis
7. Generate impact report
8. Generate test intents
9. Generate test cases

The orchestrator must block progression if a required phase has not completed.

### 8.2 Source-Grounded Parameter Extraction
For every affected route, the system must determine:
- HTTP method
- Path
- Path params
- Query params
- Request body schema
- Response schema
- Headers of interest if detectable
- Auth/dependency requirements if detectable

Parameters must come from code parsing, schema introspection, or explicit route metadata. The model must not invent them.

### 8.3 Cross-Layer Impact Expansion
The system must connect:
- Backend route handlers to request/response models
- Shared models to frontend types or client adapters
- Frontend pages/components/hooks to API calls
- Changes in auth or middleware to downstream routes/components

### 8.4 Incremental Freshness
The system must avoid stale data by supporting:
- Incremental re-indexing on file change
- Git-diff-based re-ingestion on demand
- Live fallback reads from source files when confidence is low or indexing is incomplete

### 8.5 Explainability
For each test suggestion, the system must provide:
- What change triggered the test
- Which route/component/model is affected
- What risk the test covers
- Which extracted schema or dependency path supports it

## 9. Proposed Architecture
### 9.1 High-Level Components
1. **Repo Diff Adapter**
   - Reads commit hashes, ranges, or working tree diffs
2. **Code Ingestion Layer**
   - Parses FastAPI routes and Pydantic models
   - Parses Next.js routes, components, hooks, and API clients
3. **Indexes**
   - Structured route/model/component index
   - Optional semantic search index
   - Dependency graph
4. **Analysis Tools Layer**
   - Exposes deterministic tools to the agent
5. **Orchestrator / State Machine**
   - Enforces workflow order and stop conditions
6. **Validation Layer**
   - Detects missing route params, unresolved symbols, and insufficient coverage
7. **Test Generation Layer**
   - Consumes approved analysis report only

### 9.2 Recommended Storage
- Structured metadata store for routes, models, components, and edges
- Graph representation for dependency traversal
- Optional vector index for semantic recovery only, not as primary truth

## 10. Data Model Requirements
### 10.1 Route Record
Each route record should include fields similar to:
- `framework`
- `file`
- `method`
- `path`
- `handler_name`
- `path_params`
- `query_params`
- `body_model`
- `response_model`
- `dependencies`
- `auth_requirements`
- `tags` if present

### 10.2 Model Record
Each model record should include:
- `name`
- `file`
- `fields`
- `field_types`
- `required_fields`
- `defaults`
- `nested_models`
- `validators` if detectable

### 10.3 Frontend Usage Record
Each frontend usage record should include:
- `file`
- `component_or_hook`
- `route_or_api_target`
- `method` if known
- `request_type`
- `response_type`
- `guards` or feature flags if known

### 10.4 Dependency Edge Types
At minimum support:
- `imports`
- `calls`
- `returns`
- `uses_model`
- `renders`
- `fetches`
- `depends_on_auth`

## 11. Required Agent Tools
The coding agent implementing this system must expose tools with behavior equivalent to the following.

### 11.1 Diff Tools
#### `get_commit_diff`
Input:
```json
{ "commit": "33cffa" }
```
Returns:
- changed files
- diff hunks or summary
- optional changed symbols if available

#### `get_diff_range`
Input:
```json
{ "range": "HEAD~2..HEAD" }
```
Returns equivalent diff details.

### 11.2 Repo Structure Tools
#### `get_changed_files`
Returns normalized changed file paths.

#### `get_symbols_for_file`
Input:
```json
{ "file": "backend/api/users.py" }
```
Returns route handlers, classes, functions, models, and exported symbols.

### 11.3 Backend Analysis Tools
#### `get_routes_for_file`
Input:
```json
{ "file": "backend/api/users.py" }
```
Returns all FastAPI routes defined or referenced in the file.

#### `get_route_schema`
Input:
```json
{ "route": "/users/{id}", "method": "GET" }
```
Returns source-grounded route details:
```json
{
  "path_params": ["id"],
  "query_params": [{"name": "includePosts", "type": "boolean", "required": false}],
  "body": null,
  "response_model": "UserResponse",
  "auth_requirements": ["bearer_token"]
}
```

#### `get_model_schema`
Input:
```json
{ "name": "UserResponse" }
```
Returns full model field details.

### 11.4 Frontend Analysis Tools
#### `get_frontend_consumers`
Input:
```json
{ "route": "/users/{id}", "method": "GET" }
```
Returns pages, hooks, components, and client modules that consume the route.

#### `get_component_dependencies`
Input:
```json
{ "file": "apps/web/app/users/[id]/page.tsx" }
```
Returns data-fetching hooks, child components, and key props/models.

### 11.5 Graph / Impact Tools
#### `trace_dependencies`
Input:
```json
{ "symbol": "UserResponse", "depth": 3 }
```
Returns upstream and downstream impacted nodes and edge paths.

#### `trace_route_to_ui`
Input:
```json
{ "route": "/users/{id}", "method": "GET" }
```
Returns the path from backend route to frontend consumers.

### 11.6 Validation Tools
#### `check_analysis_complete`
Input:
```json
{
  "changed_files": [],
  "affected_routes": [],
  "affected_models": [],
  "frontend_consumers": [],
  "parameter_records": []
}
```
Returns:
```json
{
  "complete": false,
  "missing": [
    "query params missing for GET /users/{id}",
    "frontend consumers unresolved for UserResponse"
  ],
  "warnings": ["auth requirements inferred from dependency and should be verified from source"]
}
```

### 11.7 Live Source Fallback Tool
#### `read_source_file`
Reads current file contents when index confidence is low or freshness is uncertain.

## 12. Orchestrator Requirements
### 12.1 State Machine
The system must behave like a state machine, not a free-form chat loop.

Required states:
- `INIT`
- `DIFF_RESOLVED`
- `SYMBOLS_EXTRACTED`
- `IMPACT_EXPANDED`
- `PARAMS_EXTRACTED`
- `VALIDATED`
- `ANALYSIS_REPORTED`
- `TEST_INTENTS_CREATED`
- `TESTS_GENERATED`

The orchestrator must reject transitions that skip required prior states.

### 12.2 Mandatory Stop Conditions
The agent must stop and continue research rather than proceed if any of the following are true:
- An affected route is missing schema details
- A changed model has unresolved downstream consumers
- A frontend route/component is linked to an affected backend route but not yet analyzed
- Validation reports missing artifacts
- Confidence is below a configured threshold

### 12.3 Minimum Investigation Requirements
Before test generation, the orchestrator must require at least:
- One diff retrieval step
- One structure lookup step for each changed file type category involved
- One dependency expansion step for every affected route or model cluster
- One schema extraction step for every affected route
- One validation step

## 13. Agent Prompt / Policy Requirements
The system prompt or policy layer must enforce the following rules:
- You are not allowed to generate tests until analysis completeness checks pass.
- You must use tools rather than assumptions to discover parameters.
- If a route or model is unresolved, continue investigation.
- Prefer source-of-truth schema extraction over semantic guesses.
- If data appears stale or confidence is low, read the source file directly.
- Separate analysis output from test generation output.

## 14. Required Intermediate Artifact
Before tests are generated, the agent must emit a structured analysis artifact similar to the following:

```json
{
  "request_scope": "test change in commit 33cffa",
  "changed_files": [],
  "changed_symbols": [],
  "affected_routes": [],
  "affected_models": [],
  "affected_frontend_components": [],
  "dependency_paths": [],
  "parameter_records": [],
  "confidence": 0.0,
  "unknowns": [],
  "validation": {
    "complete": false,
    "missing": []
  }
}
```

This artifact is the contract between analysis and test generation.

## 15. Test Generation Requirements
### 15.1 Test Intent Categories
For each affected behavior, produce explicit intents in these buckets where applicable:
- Happy path
- Validation errors
- Auth and permission behavior
- Boundary conditions
- Null/empty/default handling
- Pagination/filter/sorting parameter interactions
- Backward compatibility and schema change risks
- Loading/error/empty UI states
- Integration between frontend callsite and backend response model

### 15.2 Grounding Requirement
Every proposed test must reference at least one of:
- affected route
- affected model
- affected frontend component or hook
- dependency path from the analysis artifact

### 15.3 Example Output Shape
Each test proposal should include:
- `title`
- `type` such as unit, integration, e2e, contract, regression
- `target`
- `why`
- `preconditions`
- `inputs`
- `assertions`
- `risk`
- `source_links` back to analysis findings

## 16. Freshness and Active Development Requirements
### 16.1 Incremental Ingestion
The implementation should support:
- update-on-save or file-watch ingestion locally
- git-diff-based refresh when analyzing commits or PRs
- selective reprocessing of only changed files and impacted neighbors

### 16.2 Staleness Protection
If the index version predates the analyzed commit or working tree state, the system must:
- refresh impacted files, or
- fall back to live source reads before validation passes

### 16.3 Idempotent Updates
Re-ingesting a file must replace prior index entries for that file and rebuild affected graph edges to avoid duplicates and stale relationships.

## 17. Observability and Debugging Requirements
The system must record:
- tool calls made
- files and symbols inspected
- unresolved gaps encountered
- reasons validation failed
- final evidence used for each proposed test

This log must be available for developer debugging and tuning.

## 18. Performance Requirements
Target constraints for initial implementation:
- Commit-level impact analysis should complete fast enough for interactive use on a typical PR-sized diff
- The system should avoid full-repo re-indexing for normal requests
- The system should degrade gracefully by narrowing scope or reading source directly when the graph/index is incomplete

Exact latency targets may be set by the implementation team based on repository size.

## 19. Quality and Acceptance Criteria
### 19.1 Functional Acceptance Criteria
The system is acceptable when it can consistently:
- identify changed files from a commit or range
- identify affected routes and models beyond directly edited files
- extract real route parameters without hallucination
- identify relevant frontend consumers for affected backend routes in common cases
- block test generation if route schema details are missing
- generate tests that are traceable to the analysis artifact

### 19.2 Quality Metrics
Track at minimum:
- Route impact recall
- Parameter extraction accuracy
- Rate of hallucinated parameters
- Percentage of generated tests with explicit evidence links
- Percentage of runs blocked correctly due to incomplete analysis
- User-rated relevance of proposed tests

### 19.3 Suggested Success Thresholds
Initial target thresholds:
- 90%+ parameter extraction accuracy on benchmark commits
- Near-zero hallucinated route parameters
- 85%+ evidence-linked test proposals
- Clear reduction in missed regression scenarios versus current baseline

## 20. Benchmarking and Evaluation Plan
Create a benchmark set of representative commits covering:
- route signature changes
- model field changes
- auth dependency changes
- pagination/filter/query parameter changes
- frontend component changes consuming stable routes
- backend response changes affecting frontend rendering

For each benchmark commit, maintain expected outputs:
- affected routes
- affected models
- affected frontend consumers
- required parameter sets
- expected test intents

Use this benchmark for regression evaluation of the testing agent.

## 21. Risks and Mitigations
### Risk: Static analysis misses dynamic route or client behavior
Mitigation:
- allow live file reads
- allow confidence downgrades and explicit unknowns
- use semantic retrieval only as backup

### Risk: Agent still jumps ahead and writes tests
Mitigation:
- enforce orchestrator state machine and validation gate outside the model

### Risk: Stale index in active development
Mitigation:
- incremental ingestion, versioning, and staleness checks

### Risk: Overly broad impact causes noisy test plans
Mitigation:
- preserve dependency depth limits and confidence scoring
- distinguish direct impact from transitive impact

## 22. Implementation Phases
### Phase 1: MVP
- Commit diff ingestion
- FastAPI route and Pydantic model extraction
- Basic Next.js API/client usage extraction
- Structured analysis artifact
- Validation gate
- Test generation from approved artifact

### Phase 2: Impact Graph Expansion
- Dependency graph traversal
- Route-to-frontend linkage
- Confidence scoring
- Better unresolved-gap reporting

### Phase 3: Freshness and Observability
- Incremental ingestion
- file-watch or diff-based refresh
- detailed evidence logs
- benchmark harness

### Phase 4: Advanced Quality Improvements
- contract-test suggestions
- reranking of affected artifacts
- automatic test selection/prioritization
- CI integration

## 23. Explicit Build Instructions for the Coding Agent
The coding agent implementing this PRD must:
1. Build the system as an orchestrated workflow, not a single prompt.
2. Separate analysis from generation into distinct stages or agents.
3. Enforce validation before test generation.
4. Treat route schema extraction as source-of-truth.
5. Prefer deterministic tools and structured artifacts over long free-form reasoning.
6. Include logging and a benchmark harness from the start.

## 24. Final Principle
The implementation should optimize for **correctness of impact discovery before creativity of test suggestion**.

If the system is unsure, it should return unresolved gaps and continue analysis rather than confidently invent tests.
