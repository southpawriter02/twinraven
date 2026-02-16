# TwinRaven

**Memory-Grounded Tool Synthesis**

> _Every dawn Huginn and Muninn fly out over the wide world; I fear for Huginn that he may not return, yet more anxious am I for Muninn._
> — Grímnismál, Poetic Edda

TwinRaven is a framework for agents that watch themselves work, and then get better at it. It observes tool usage patterns over time and synthesizes new composite tools from frequently-chained operations. Named for Odin's two ravens, Huginn (thought) and Muninn (memory), the system implements a continuous loop where memory informs reasoning and reasoning reshapes how memory is collected.

The result: self-optimizing workflows that sharpen every time they run. Your agents literally learn from their own habits.

---

## The Huginn-Muninn Loop

In Norse mythology, Odin sends his two ravens out at dawn. They fly across the world, observe everything, and return at dusk to whisper what they've learned into his ear. Huginn sees meaning. Muninn remembers everything. Neither is useful without the other.

(Odin had the right idea. He just didn't have Python.)

TwinRaven applies this same architecture to agent tool use:

```
                    ┌─────────────────────────┐
                    │      Agent Runtime       │
                    │   (tool invocations)     │
                    └────────┬────────────────┘
                             │
                             ▼
                ┌────────────────────────┐
                │   Muninn (Memory)      │
                │                        │
                │  - Telemetry capture   │
                │  - Sequence logging    │
                │  - Context windows     │
                │  - Outcome tracking    │
                └────────────┬───────────┘
                             │
                             ▼
                ┌────────────────────────┐
                │   Huginn (Thought)     │
                │                        │
                │  - Pattern mining      │
                │  - Chain detection     │
                │  - Tool generation     │
                │  - Validation          │
                └────────────┬───────────┘
                             │
                             ▼
                ┌────────────────────────┐
                │   Synthesized Tools    │
                │                        │
                │  - Composite actions   │
                │  - Optimized params    │
                │  - Built-in error      │
                │    handling            │
                └────────────┬───────────┘
                             │
                             └──────► fed back into Agent Runtime
```

The loop never sleeps. Every synthesized tool, once deployed, generates its own telemetry. Muninn observes how the new tool is used; Huginn considers whether it should be refined, split, or composed further. Tools don't just exist. They _evolve._

---

## Core Concepts

### Muninn — The Memory Layer

Muninn is the faithful scribe. No interpretation, no editorializing; just meticulous, obsessive record-keeping of every tool invocation with enough context to reconstruct what the agent was thinking and doing.

The Edda's anxiety about losing Muninn is well-placed. Without memory, thought has nothing to chew on.

**What Muninn records:**

| Field            | Description                                      |
| ---------------- | ------------------------------------------------ |
| `tool_id`        | Which tool was called                            |
| `input_params`   | The arguments passed                             |
| `output_summary` | A compressed representation of the result        |
| `timestamp`      | When the call occurred                           |
| `session_id`     | Which agent session this belongs to              |
| `predecessor`    | The tool call that immediately preceded this one |
| `successor`      | The tool call that immediately followed          |
| `outcome`        | Whether the broader task succeeded or failed     |
| `latency_ms`     | Execution time                                   |

**Storage model:** Muninn writes to an append-only event log, the one true timeline. Each entry is a structured record linking a tool invocation to its temporal neighbors. All downstream analysis reads from this log. If it's not in the log, it didn't happen.

```
MuninnEvent {
  event_id:    uuid
  session_id:  string
  tool_id:     string
  input_hash:  string      // deterministic hash of input params
  input_params: object
  output_summary: string   // LLM-compressed output (not raw)
  predecessor: uuid | null
  successor:   uuid | null
  timestamp:   iso8601
  latency_ms:  int
  outcome:     success | failure | partial
  tags:        string[]    // user-defined or auto-extracted
}
```

### Huginn — The Thought Layer

Huginn reads what Muninn has dutifully stored and goes looking for structure: the patterns hiding in the noise. Its job is to find tool chains that repeat often enough and reliably enough to justify forging them into a single composite tool.

Think of it as refactoring, but the codebase is your agent's behavior and the compiler is an LLM.

**Phase 1: Pattern Mining**

Huginn runs sequence mining algorithms against the event log:

- **PrefixSpan** — Finds frequent sequential patterns in tool invocation sequences. Given a minimum support threshold, it extracts all tool chains that appear across multiple sessions.
- **GSP (Generalized Sequential Patterns)** — Adds time constraints to the mining. A chain of `search → read → summarize` only counts if the calls happen within a configurable time window, filtering out coincidental co-occurrence. (Correlation ≠ causation, even for ravens.)
- **Session-Aware Windowing** — Patterns are mined within session boundaries. A tool chain that spans two unrelated sessions is noise, not signal.

The output is a ranked list of candidate chains:

```
CandidateChain {
  tools:       [tool_id, tool_id, ...]
  support:     float    // fraction of sessions containing this chain
  confidence:  float    // P(successor | predecessor) for each link
  avg_latency: int      // total chain execution time
  failure_rate: float   // how often the chain ends in a failed outcome
  sample_ids:  [event_id, ...]  // representative instances
}
```

**Phase 2: Tool Generation**

For each candidate chain that clears the bar (minimum support, minimum confidence, maximum failure rate), Huginn generates a composite tool definition.

This is where the LLM earns its keep:

1. **Merge parameters** — Analyze the input parameters across all tools in the chain. Identify which outputs from step N become inputs to step N+1 (internal wiring) and which need to be exposed as parameters of the composite tool.
2. **Define error handling** — Examine the failure cases from sample instances. Generate retry logic, fallback paths, or early-exit conditions based on observed failure modes.
3. **Optimize execution** — Determine whether any steps can run in parallel rather than sequentially. Identify steps that can be skipped under certain input conditions.
4. **Write the specification** — Produce a tool definition in a standard format (function schema, description, parameter types) along with the implementation that orchestrates the underlying tools.

```
SynthesizedTool {
  tool_id:       string          // auto-generated, human-reviewable name
  description:   string          // what this composite tool does
  parameters:    JSONSchema       // merged external parameters
  internal_wiring: object        // how intermediate outputs connect
  steps:         [StepDef, ...]  // ordered tool calls with conditions
  error_strategy: object         // retry, fallback, abort rules
  source_chain:  CandidateChain  // provenance
  version:       int
  status:        draft | testing | promoted | retired
}
```

**Phase 3: Validation**

Synthesized tools don't get a free pass. Huginn replays historical sessions containing the source chain and checks whether the composite tool would have produced equivalent results:

- **Functional equivalence** — Given the same initial inputs, does the composite tool produce the same final outputs as the original chain?
- **Error parity** — Does it handle failure cases at least as well as the original sequence?
- **Latency improvement** — Is it actually faster, or does the orchestration overhead negate the gains? (If the "optimization" is slower, it's not an optimization. Radical concept.)

Only tools that pass validation are promoted to production. Failed candidates are logged with the reason for failure so Huginn can learn what kinds of chains synthesize well, and which ones were wishful thinking.

---

## The Feedback Loop

Here's where things get _recursive,_ and if that word excites you, you're in the right place.

Once a composite tool is promoted, it re-enters the observation cycle:

1. **Muninn logs its usage** just like any other tool: inputs, outputs, latency, outcomes. No special treatment, no VIP lane.
2. **Huginn can compose it further.** A synthesized tool can become part of a new candidate chain and be folded into an even higher-order composite. It's tools all the way down.
3. **Huginn can decompose it.** If a synthesized tool starts failing more often than its constituent parts did individually, Huginn flags it for retirement and falls back to the original chain. No ego, no sunk-cost fallacy.
4. **Drift detection.** Muninn tracks whether the usage patterns that justified a tool's creation are still present. If the underlying workflow changes and the composite tool sees declining use, it's a candidate for retirement.

This is the Huginn-Muninn loop: observation feeds synthesis, synthesis changes what is observed, and the system continuously tightens around the workflows that actually matter.

```
Session 1:  search → read → summarize  (Muninn records)
Session 2:  search → read → summarize  (Muninn records)
  ...
Session N:  search → read → summarize  (Muninn records)
            ───────────────────────────
            Huginn: "this chain has 87% support, 94% confidence"
            Huginn: generates `research_and_summarize` tool
            Huginn: validates against historical sessions
            Huginn: promotes to production
            ───────────────────────────
Session N+1: research_and_summarize    (Muninn records the composite)
Session N+2: research_and_summarize → draft_report (Muninn records)
  ...
            Huginn: "new chain detected..."
```

---

## Architecture

```
twinraven/
├── muninn/                  # Memory layer
│   ├── collector.py         # Telemetry capture hooks
│   ├── event_log.py         # Append-only event store
│   ├── schemas.py           # MuninnEvent and related types
│   └── exporters/           # Log export formats (JSON, Parquet, OTLP)
├── huginn/                  # Thought layer
│   ├── mining/
│   │   ├── prefixspan.py    # PrefixSpan implementation
│   │   ├── gsp.py           # Generalized Sequential Patterns
│   │   └── candidates.py    # Chain ranking and filtering
│   ├── synthesis/
│   │   ├── generator.py     # LLM-based tool generation
│   │   ├── merger.py        # Parameter merging logic
│   │   └── templates/       # Tool definition templates
│   └── validation/
│       ├── replay.py        # Historical session replay
│       ├── equivalence.py   # Output comparison
│       └── promotion.py     # Lifecycle management
├── tools/                   # Synthesized tool registry
│   ├── registry.py          # Tool storage and lookup
│   ├── lifecycle.py         # draft → testing → promoted → retired
│   └── generated/           # Auto-generated tool definitions
├── integrations/            # Framework adapters
│   ├── langchain.py         # LangChain Tools adapter
│   ├── crewai.py            # CrewAI adapter
│   └── base.py              # Generic agent integration
├── cli/                     # Command-line interface
│   ├── analyze.py           # Usage analysis commands
│   ├── approve.py           # Manual tool approval workflow
│   ├── inspect.py           # Explore event logs and chains
│   └── main.py              # CLI entrypoint
└── config.py                # Thresholds, storage backends, LLM config
```

---

## Delivery Format

TwinRaven is designed as an **agent framework extension** that plugs into your existing agent stack rather than replacing it. Think of it as a parasitic optimization layer. (The good kind of parasitic. Symbiotic. Whatever, it makes your agents better.)

### Framework Integration

**LangChain:**

```python
from twinraven.integrations.langchain import TwinRavenToolkit

# Wrap existing tools with Muninn observation
toolkit = TwinRavenToolkit(
    tools=existing_tools,
    event_store="sqlite:///muninn.db"
)

# Later, retrieve synthesized tools alongside originals
all_tools = toolkit.get_tools(include_synthesized=True)
```

**CrewAI:**

```python
from twinraven.integrations.crewai import TwinRavenCrewTools

crew_tools = TwinRavenCrewTools(
    tools=existing_tools,
    config=twinraven_config
)
```

**Custom agents:**

```python
from twinraven.muninn import Collector
from twinraven.huginn import Synthesizer

collector = Collector(store="sqlite:///muninn.db")

# In your agent loop
with collector.observe(session_id="abc"):
    result = my_tool.run(params)
    # Muninn records automatically

# Periodically run Huginn
synthesizer = Synthesizer(store="sqlite:///muninn.db")
new_tools = synthesizer.run(
    min_support=0.3,
    min_confidence=0.8,
    max_failure_rate=0.1
)
```

### CLI

```bash
# Analyze tool usage patterns
twinraven analyze --store muninn.db --min-support 0.3

# Inspect a specific candidate chain
twinraven inspect chain <chain-id> --show-samples

# Review and approve a synthesized tool
twinraven approve <tool-id> --dry-run
twinraven approve <tool-id> --promote

# Export event log
twinraven export --format parquet --output events.parquet

# Retire a synthesized tool
twinraven retire <tool-id> --reason "workflow changed"
```

---

## Configuration

```yaml
# twinraven.yaml
muninn:
  store: sqlite:///muninn.db # Event log backend
  output_compression: true # LLM-compress tool outputs before storing
  max_output_length: 500 # Truncate output summaries
  retention_days: 90 # Auto-prune old events

huginn:
  mining:
    algorithm: prefixspan # prefixspan | gsp
    min_support: 0.3 # Minimum session frequency
    min_confidence: 0.8 # Minimum transition probability
    max_chain_length: 6 # Ignore chains longer than this
    time_window_seconds: 300 # GSP: max gap between steps
  synthesis:
    llm_model: claude-sonnet-4-20250514 # Model for tool generation
    require_approval: true # Human-in-the-loop before promotion
    max_parallel_steps: 3 # Parallelism limit in generated tools
  validation:
    min_replay_sessions: 10 # Minimum historical sessions to test
    equivalence_threshold: 0.95 # Output similarity required
    max_latency_regression: 1.2 # Composite can be at most 1.2x slower

tools:
  registry: ./tools/generated # Where synthesized tools are stored
  auto_retire_after_days: 30 # Retire unused tools after 30 days
  max_active_tools: 50 # Cap on simultaneous synthesized tools
```

---

## Design Principles

**Muninn is append-only.** The event log is never modified after write. Corrections are new events, not mutations. This guarantees that Huginn always works from a complete history and that validation replay is deterministic. (Version control for your agent's memory. `git blame` for ravens.)

**Huginn is conservative.** The default thresholds are deliberately high. It is better to miss a valid composite tool than to promote a bad one. Every synthesized tool carries the overhead of another abstraction, and it must earn its place. No participation trophies.

**Humans approve.** By default, `require_approval: true`. Huginn proposes, a human disposes. The CLI provides inspection commands so reviewers can see exactly which sessions, chains, and failure modes informed a synthesized tool before giving it the thumbs up. Trust, but verify.

**Tools are mortal.** Every synthesized tool has a lifecycle. If it stops being used, it gets retired automatically. If it starts failing, Huginn flags it. There is no tool graveyard of unused abstractions accumulating forever, unlike that `utils.py` file we all pretend doesn't exist.

**The loop is the product.** Individual components (telemetry, mining, generation) exist elsewhere. You can find each piece in a dozen different repos. The value of TwinRaven is the _closed loop:_ Muninn feeds Huginn, Huginn changes what Muninn observes, and the system tightens around the workflows that actually matter. The ravens fly together, or they don't fly at all.

---

## License

GPL-3.0 — see [LICENSE](LICENSE).
