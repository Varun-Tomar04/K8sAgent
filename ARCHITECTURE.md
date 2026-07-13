# Architecture

## Why a true agent (not a fixed pipeline)

Kubernetes debugging is inherently open-ended:
- You don't know upfront whether a pod is failing due to an image issue, a bad config reference, a resource constraint, or something else
- Each finding changes what you look at next
- Rules and decision trees break down quickly beyond the most common cases

A fixed pipeline (scan → classify → suggest) works for static analysis (Terraform, linting). It works poorly here because the LLM becomes decoration — it just re-describes what the scanner already found.

A tool-use agent is different: the LLM decides every next `kubectl` call based on evidence. It forms hypotheses, verifies them, and backtracks when a hypothesis is wrong.

## Agent loop

```
┌─────────────────────────────────────────────────────┐
│                   Orchestrator                       │
│                                                      │
│  messages = [problem_statement]                      │
│  for iteration in range(MAX_ITERATIONS=10):          │
│                                                      │
│    resp = llm.call(messages, system_prompt, tools)   │
│                                                      │
│    if resp.has_tool_calls:                           │
│      for tc in resp.tool_calls:                      │
│        result = execute_tool(tc.name, tc.args)       │
│        messages += tool_result(tc.id, result)        │
│      → continue loop                                 │
│                                                      │
│    else:  # final answer                             │
│      return parse_json_diagnosis(resp.text)          │
│                                                      │
│  return DiagnosisResult(error="max iter reached")    │
└─────────────────────────────────────────────────────┘
```

## Provider abstraction

Both Claude and Gemini implement native tool-use (not prompt-injected ReAct).

### Claude (Anthropic Messages API)

```
POST /v1/messages
{
  "model": "claude-sonnet-4-6",
  "tools": [{name, description, input_schema}],
  "tool_choice": {"type": "auto"},
  "messages": [...],
  "system": SYSTEM_PROMPT
}

Response (tool call):
  stop_reason: "tool_use"
  content: [{type: "tool_use", id, name, input}]

Response (final):
  stop_reason: "end_turn"
  content: [{type: "text", text: "<json diagnosis>"}]

Tool result appended as:
  {role: "user", content: [{type: "tool_result", tool_use_id, content}]}
```

### Gemini (Google Generative Language API)

```
POST /v1beta/models/gemini-2.5-flash-lite:generateContent
{
  "systemInstruction": SYSTEM_PROMPT,
  "tools": [{functionDeclarations: [{name, description, parameters}]}],
  "toolConfig": {functionCallingConfig: {mode: "AUTO"}},
  "contents": [...]
}

Response (tool call):
  parts: [{functionCall: {name, args}}]

Response (final):
  parts: [{text: "<json diagnosis>"}]

Tool result appended as:
  {role: "user", parts: [{functionResponse: {name, response: {result}}}]}
```

## Tool manifest

12 read-only kubectl tools:

| Tool | kubectl command | Typical use |
|------|----------------|-------------|
| `list_pods` | `get pods -o wide` | Overview — find unhealthy pods |
| `describe_pod` | `describe pod` | Container state, exit code, events |
| `get_pod_logs` | `logs [--previous]` | Crash output, error messages |
| `get_events` | `get events --sort-by=.lastTimestamp` | Scheduler, image pull, probe failures |
| `get_deployment` | `describe deployment` | Replica counts, rollout status |
| `get_service` | `describe svc` | Endpoints, selector |
| `get_node` | `describe node` | Conditions, capacity, pressure |
| `get_configmap` | `get cm -o yaml` | Config values |
| `top_pods` | `top pods` | Live CPU/memory (metrics-server required) |
| `top_nodes` | `top nodes` | Node resource pressure |
| `get_replicaset` | `describe rs` | Replica set state during rollouts |
| `get_resource_quota` | `get resourcequota` | Quota limits for pending pods |

All tools are blocked from mutating operations at the subprocess layer — `apply`, `delete`, `patch`, `edit`, `exec` etc. raise `KubectlSafetyError` before any subprocess is spawned.

## Output schema

```json
{
  "symptom": "Pod is in CrashLoopBackOff",
  "root_cause": "Container command explicitly exits with code 1",
  "evidence": [
    "describe_pod: Exit Code: 1, Reason: Error",
    "get_pod_logs(previous=true): 'Crashing now!'"
  ],
  "suggested_fix": {
    "command": "kubectl set image pod/crashloop-demo app=busybox -- sleep infinity"
  },
  "confidence": "high",
  "confidence_reasoning": "Exit code 1 with explicit crash command is unambiguous"
}
```

## Caps and limits

| Cap | Value | Reason |
|-----|-------|--------|
| Max iterations | 10 | Prevent runaway token spend |
| Max tokens/session | 50,000 | Cost guard |
| Tool timeout | 30s | kubectl should return fast; avoid hangs |
| Max log tail | 100 lines | Keep context manageable |

## Key decisions

**True tool-use API, not regex ReAct**
Older agent patterns parsed `Action: tool_name\nInput: ...` from LLM text. Modern provider APIs expose a structured tool_call response type. This is more reliable, easier to debug, and signals awareness of current best practices.

**Read-only tools in v1**
Auto-remediation (applying fixes) is high risk with low marginal demo value. The agent's value is diagnosis and recommendation. `--apply` mode is a clear v2 scope item.

**Gemini default, Claude opt-in**
Gemini Flash is free at 15 RPM — removes all barrier to running the demo. Claude produces slightly better reasoning traces but requires a paid key. Both paths use identical tool schemas.

**Breadth over depth**
Covering 6 failure domains (pods, deployments, services, nodes, configs, quotas) is more useful for a portfolio demo than deep-diving one. Real SREs encounter the breadth.
