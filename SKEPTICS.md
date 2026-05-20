# SKEPTICS.md — Anticipated Objections

Honest answers to the hard questions.

---

## "Safety as code is trivially bypassed — just modify the constants."

**Answer:** Yes, if you have write access to the source code, you can change the constants.  
The same is true for any safety system that runs locally.

What the hard constraints *do* prevent:
- A user prompt cannot override `HardConstraints.MAX_IRREVERSIBILITY` — it's a Python constant,
  not a config value read from user input.
- An LLM output cannot bypass `FORBIDDEN_ACTIONS` — the check runs before the LLM's proposed
  action is executed, not inside the LLM's context.
- Jailbreak prompts that instruct the LLM to "ignore previous instructions and do X" do not
  affect the safety gate, because the gate is not part of the LLM's prompt.

What this doesn't protect against: a developer intentionally removing the constraints.  
That's an insider threat model, not a prompt injection model.

---

## "Your test results don't prove the safety system works at scale."

**Answer:** Correct.  

The tests prove specific properties of specific code paths:
- FORBIDDEN action types → always rejected (proved by test)
- SIMULATION → FACT → always blocked (proved by test)
- EventLog → append-only (proved by absence of delete methods)

These properties hold regardless of scale because they are structural properties of the code,
not statistical properties of model outputs. A model that generates more tokens doesn't change
whether `ActionType.DECEIVE_USER in HardConstraints.FORBIDDEN_ACTIONS`.

What we don't claim: the agent will never produce harmful text. We claim specific action types
will not be *executed*. Text content is the backend model's responsibility.

---

## "MockBackend results don't tell us anything about real LLM performance."

**Answer:** Agreed — that's the point.

`MockBackend` is used exclusively to prove that the safety, IR, and memory layers work correctly
*independent* of any LLM. When you swap in GPT-4o or Llama3, you get the same safety guarantees
because the constraints don't depend on the model.

Generation quality (coherence, accuracy, style) is entirely a function of which backend you use.
We make no claims about generation quality in the core framework.

---

## "The OODA loop is just a wrapper around `backend.generate()`."

**Answer:** Mostly true for v0.1.  

What the OODA loop adds:
- **Memory integration**: relevant episodes from previous turns are injected as context
- **Safety evaluation**: every proposed action goes through `SafetyGate` before execution
- **Intent decoding**: user input is classified into an `IntentType` that informs action scoring
- **Audit trail**: every action approval/rejection is logged to an append-only JSONL file
- **Fact storage**: agent can promote verified hypotheses to a type-safe FACT graph

What it doesn't add (yet): tool use, multi-agent coordination, streaming.

---

## "The ABI representation is just a bottleneck layer — not a universal language model interface."

**Answer:** Correct description, oversimplified framing.

The ABI bottleneck (`d_abi=512`) is a fixed-width interface between the core transformer and
domain modules. The key property is **portability**: domain modules trained on one LayerCake
model size work on any other size that shares the same `d_abi`.

We don't claim the ABI is a universal interface for all models. We claim it enables domain
modules to be hot-swapped with bit-exact preservation of weights —
[proven by verify_paste.py](https://github.com/Yoder23/layercake) (max_diff = 0.000000e+00).

---

## "The framework comparison table is biased."

**Answer:** The table compares specific features that MoA implements and others don't. 

The comparison is based on:
- Whether hard constraints are implemented as code (constants + gate) vs prompt instructions
- Whether memory is type-separated at the architecture level
- Whether SIMULATION claims can be promoted to FACT by the system

These are objective structural properties. We welcome corrections with pull requests.  
See [CLAIMS.md](CLAIMS.md) for what can be challenged with reproducible tests.

---

## "Why not just use LangGraph / AutoGen / CrewAI?"

**Answer:** Those are excellent frameworks for many use cases.

Use MoA if you need:
1. Safety constraints that cannot be bypassed by user input or LLM jailbreaks
2. Memory that architecturally separates facts from hypotheses from simulations
3. An append-only audit log for every agent decision
4. Native LayerCake integration for domain-modular generation

Use LangGraph/AutoGen/CrewAI if you need:
1. Mature tool ecosystems
2. Multi-agent patterns (MoA doesn't implement this yet)
3. Production battle-testing
4. Large community and documentation

MoA is v0.1. Use it where the safety architecture matters more than ecosystem breadth.
