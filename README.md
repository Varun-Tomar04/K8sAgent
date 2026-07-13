# k8s-triage-agent

A CLI tool that debugs broken Kubernetes workloads for you. Point it at a namespace, and an LLM decides which `kubectl` commands to run, reads the output, and keeps digging until it can tell you what's actually wrong — with evidence and a suggested fix.

```
$ python agent.py diagnose --namespace triage-demo

🔍 k8s-triage-agent  GEMINI
   Investigating namespace triage-demo
──────────────────────────────────────────────────────────────
  → list_pods(namespace='triage-demo')
    NAME              READY   STATUS             RESTARTS
    crashloop-demo    0/1     CrashLoopBackOff   5
  → describe_pod(name='crashloop-demo', namespace='triage-demo')
    ...Exit Code: 1...
  → get_pod_logs(name='crashloop-demo', previous=True)
    Crashing now!

━━━ DIAGNOSIS ━━━
Symptom:    Pod crashloop-demo is in CrashLoopBackOff
Root Cause: Container command always exits with code 1
Evidence:
  • describe_pod: Last State Terminated, Exit Code: 1
  • get_pod_logs(previous=true): "Crashing now!"

━━━ CONFIDENCE ━━━
● HIGH  Exit code 1 with explicit crash command is unambiguous
```

## Why I built this

My previous project was a Terraform static analysis gate — a fixed pipeline, because security gates need to be deterministic. This is the opposite problem. When a pod breaks, you don't know upfront if it's the image, the config, a resource limit, or a probe. Each thing you find changes what you check next. That's what an agent is actually good for, so I built one.

The loop is written by hand, no LangChain. Both Claude and Gemini are supported through their native tool-calling APIs, and the whole thing is around 1,500 lines across 9 files, so it's small enough to read in one sitting.

## How it works

The LLM gets a problem statement ("namespace X is broken") and 12 read-only kubectl tools. It picks a tool, we run the command, feed the output back, and it picks again — until it has enough evidence to return a JSON diagnosis. Capped at 10 iterations and 50k tokens so it can't run away.

```
agent.py                     CLI entry
├── core/
│   ├── orchestrator.py      the agent loop
│   ├── llm.py               ClaudeProvider / GeminiProvider
│   └── prompts.py           system prompt + result dataclasses
├── tools/
│   ├── definitions.py       12 tool schemas (one list, both providers)
│   ├── executors.py         tool name → kubectl function
│   └── kubectl.py           subprocess wrapper, read-only, 30s timeout
└── output/
    ├── console.py           live streaming terminal output (rich)
    └── json_out.py          --json mode
```

Safety is enforced at the subprocess layer, not in the prompt: `kubectl.py` blocks all 20 mutating subcommands (`apply`, `delete`, `patch`, `exec`, ...) before anything is executed, and runs with `shell=False`. Even if the model hallucinates a destructive command, it gets a "blocked" string back as the tool result and has to try something else.

## Setup

You'll need Python 3.11+, kubectl, minikube, and Docker.

```bash
git clone https://github.com/Varun-Tomar04/K8sAgent
cd k8s-triage-agent
python -m venv .venv
source .venv/bin/activate        # Windows bash: source .venv/Scripts/activate
pip install -r requirements.txt

cp .env.example .env
# add your GEMINI_API_KEY — free at https://aistudio.google.com/apikey
```

Then deploy the demo scenarios and run it:

```bash
bash scenarios/setup.sh
python agent.py diagnose --namespace triage-demo
```

## Usage

```bash
# whole namespace — the agent finds unhealthy pods itself
python agent.py diagnose --namespace triage-demo

# specific pod
python agent.py diagnose --pod crashloop-demo --namespace triage-demo

# use Claude instead of Gemini
python agent.py diagnose --namespace triage-demo --llm-provider claude

# give it a hint
python agent.py diagnose --namespace triage-demo --context-hint "broke after deploy at 14:00"

# machine-readable output, for piping into other tools
python agent.py diagnose --namespace triage-demo --json
```

## Demo scenarios

`scenarios/` has eight intentionally broken workloads, from trivial to genuinely annoying:

| Pod | What's wrong |
|-----|--------------|
| `imagepull-fail` | image tag doesn't exist |
| `oom-victim` | memory limit 10Mi, workload needs 50Mi |
| `missing-config` | references a ConfigMap that doesn't exist |
| `liveness-looper` | liveness probe hits a 404 endpoint |
| `crashloop-demo` | container just exits 1 |
| `resource-hog` | requests 1000 CPU cores, unschedulable |
| `backend-svc` | service selector doesn't match pod labels |
| `stuck-rollout` | bad image + `maxUnavailable: 0`, rollout never finishes |

The last two are the interesting ones — the selector mismatch produces no errors anywhere (the pod is healthy, the service exists, only the endpoints list is empty), and the stuck rollout is two bugs interacting. Those are the cases where an agent beats a rules engine.

## Limits

- Read-only. It diagnoses, it doesn't fix. That's intentional, not a TODO.
- No CNI/DNS/RBAC deep dives — the 12 tools don't cover them, and it'll say "low confidence" rather than guess.
- Run-once, no watch mode. Use real monitoring for detection; use this for triage.

## Ideas for later

- Slack bot — `run_investigation()` already returns a serializable result, so this is ~50 lines of glue
- `--apply` mode behind an explicit confirmation
- A Prometheus tool for things kubectl can't see

## More docs

- [ARCHITECTURE.md](ARCHITECTURE.md) — tool schemas, API formats, decision log
- [scenarios/README.md](scenarios/README.md) — scenario details and teardown
