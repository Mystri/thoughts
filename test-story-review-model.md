# A Two-Ended Model for Test Design and Review

Status: working design note

Context: developed while studying TorchTitan and TorchTitan-NPU tests, but intended to describe a
general method for designing and reviewing change-oriented test evidence.

Source working notes:

- [TorchTitan-NPU testcase review policy draft](working-notes/testcase-review-policy-draft.md)
- [TorchTitan-NPU test-story simulation progress report](working-notes/test-story-simulations-progress.md)

These snapshots preserve the discussion, PR-439 analysis, representative commit simulations, and
intermediate models from which this distilled note was derived. Their status statements refer to
policy/skill integration in the original TorchTitan-NPU workspace.

## The problem

A test suite can have high line coverage, many parameterized cases, and sensible directory
ownership while still failing to prove the behavior promised by a change.

The central review question is not:

> How many tests touch this change?

It is:

> Why does this evidence establish the effective intention of the change, and which plausible
> incorrect implementation would it reject?

A category matrix helps locate tests. A coverage calculator helps find unexecuted code. Neither
constructs the proof argument connecting a PR idea to observable behavior.

## Core objects

Keep the following objects separate:

### PR intention

What must be true if the change succeeds. Derive the effective intention from more than the PR
description:

- issue, reproducer, or user-visible problem;
- promised behavior;
- implementation diff and changed callsites;
- enable, disabled, default, invalid, and unsupported behavior;
- existing compatibility and preservation contracts;
- supported models, shapes, dtypes, topologies, backends, and checkpoint versions;
- numerical, performance, and memory claims.

If the prose and diff disagree, report the disagreement instead of redefining the intention around
the implementation.

### Required proposition

An independently falsifiable claim that must be established as part of the PR intention. Required
propositions are organized into proof stories.

### Literal testcase proposition

The narrowest statement directly supported by a testcase's setup, production operation, and
assertions.

For testcase `T`, call this `L(T)`. It describes only:

- concrete or parameterized preconditions actually constructed;
- production operation actually executed;
- observations actually asserted;
- exact tested domain and evidence layer.

It does not include presumed author motivation, unexecuted downstream effects, or universal scope
unsupported by the cases.

### Generalized testcase contract

A broader contract suggested by testcase names, parameterization, documentation, or surrounding
code. This is an inference, not a literal observation, and should be labeled as such.

Three parameter rows do not literally prove an `if and only if` statement for every possible
configuration. The literal proposition should say "for the tested cases" unless the domain is
genuinely exhaustive or generated systematically.

### Test story

A causal proof argument leading to one independently falsifiable terminal claim. A story connects
producer contracts, representations, transformations, runtime plumbing, consumers, and observable
outcomes.

One PR may need several test stories.

### PR evidence case

The complete, connected bundle of proof stories, testcase observations, numerical artifacts,
checkpoint results, and performance evidence used to evaluate the PR intention.

## The two-ended workflow

Test drafting and test review enter the same model from opposite ends.

```text
Top down: PR idea -> required proof stories -> verifiable chapters ----\
                                                                   reconcile
Bottom up: testcase code -> literal propositions -> evidence stories --/
```

The two sides should initially be developed independently:

- the PR description must not cause a reviewer to overstate what assertions prove;
- existing tests must not redefine required behavior around whatever happens to be covered.

## Top-down requirement tunnel

```text
PR idea
  -> effective intention
  -> independently falsifiable terminal claims
  -> one or more proof stories
  -> small verifiable chapters
  -> required propositions and evidence conditions
```

Each chapter should identify:

- one primary behavioral proposition;
- controlled stimulus or precondition;
- independently observable result;
- smallest discriminating scenario set;
- plausible fault it must reject;
- required CPU, accelerator, multi-rank, checkpoint, numerical, or performance layer;
- input and output handoffs to adjacent chapters;
- default, preservation, boundary, and unsupported branches that apply.

"Small" does not mean that every chapter maps to one testcase or a few scalar inputs. It means the
chapter is independently falsifiable and its stimuli and observations are controllable. Do not
decompose a collective, convergence, or performance claim until a mock appears to prove something
that only the real execution layer can establish.

## Bottom-up evidence tunnel

For every drafted or existing testcase:

```text
testcase implementation
  -> literal proposition
  -> proposition review
  -> retain, revise, supplement, split, or remove
  -> group accepted propositions into causal evidence stories
```

Extract the literal proposition before assigning the testcase a grand narrative role. Only after
the literal statement is stable should the review add:

- inferred generalized contract;
- activation evidence;
- oracle quality;
- rejected fault;
- semantic height;
- evidence-layer limitations;
- prior producer contract consumed;
- later claim enabled;
- proposed role in a larger story.

Never improve a weak testcase by writing a stronger description than its assertions support.
Stronger propositions require stronger executable evidence.

## Proposition review gate

| Gate | Review question |
| --- | --- |
| Extractability | Can two reviewers derive materially the same literal proposition from the code? |
| Fidelity | Does the description avoid stronger verbs, scope, or quantifiers than the assertions? |
| Focus | Is there one primary behavioral proposition, even if several causal assertions support it? |
| Validity | Do activation, observation, and oracle actually establish the literal proposition? |
| Relevance | Does it map to a required claim, preservation contract, or named risk? |
| Discrimination | Which plausible wrong implementation would fail? |
| Layer adequacy | Can this CPU, mock, single-rank, or device layer observe the required behavior? |
| Story connection | Which prior fact does it consume, and which later claim may rely on it? |

A testcase can have a precise and useful literal proposition while remaining insufficient for the
PR. A helper test may protect a seam without proving that a runtime feature is activated.

## Testcase review outcomes

| Outcome | Meaning |
| --- | --- |
| Retain as core evidence | The literal proposition establishes a required claim at the correct layer |
| Retain as supporting evidence | It protects a useful seam but needs a higher-layer bridge |
| Narrow or rename | The test is useful but its name or description overclaims |
| Strengthen | The intended proposition is suitable but assertions or scenarios are too weak |
| Supplement | Keep the test and add another handoff or evidence layer |
| Split | Unrelated primary propositions make purpose or failures ambiguous |
| Remove | The proposition is redundant, irrelevant, circular, or protects an implementation accident |

## Story decomposition

Do not move directly from reviewed propositions to one ordered story. That assumes one PR has one
causal narrative.

Instead:

```text
reviewed propositions
  -> identify independent terminal claims
  -> group propositions into proof stories
  -> establish bridges between stories
  -> compare the complete story bundle with the PR intention
```

Propositions belong in one story when they advance the same terminal claim through a real causal
dependency. Living in the same file, module, feature, or commit is not enough.

A useful split is exposed when two claims:

- can pass or fail independently;
- use different oracles;
- require different execution layers;
- do not consume one another's output;
- answer different reviewer questions.

For example, configuration activation and numerical equivalence are separate stories:

- activation asks which implementation the production runtime executes;
- equivalence asks whether that implementation preserves required values.

They still need a bridge. The numerical story must exercise an implementation constructed through
the production activation path, or independently prove that it represents the same path.

Performance usually forms another story because it has a different baseline, oracle, measurement
protocol, and failure mode from numerical correctness.

## Meeting in the middle

Reconcile required chapters with literal evidence propositions. This is graph matching, not
test-count matching, and it is not necessarily one-to-one:

- one required chapter may need several tests or artifacts;
- one causally focused test may make several inseparable observations for one chapter;
- one test may support multiple stories, but its proposition cannot be counted as stronger on its
  second use;
- adjacent chapters must use compatible producer and consumer representations.

A match is sufficient only when the evidence:

- establishes the required proposition over the necessary scenario domain;
- activates the claimed path;
- uses an independent and adequate oracle;
- rejects the relevant fault;
- runs at an execution layer capable of observing the claim;
- connects changed handoffs using real producer output or a checked contract fixture;
- protects applicable default, preservation, and boundary behavior.

## Reconciliation outcomes

| Middle result | Interpretation | Action |
| --- | --- | --- |
| Required chapter and evidence match | The chapter is supported at the stated scope | Retain and connect it |
| Required chapter has no evidence | Promised behavior or risk is unprotected | Draft a test or request an artifact |
| Evidence is weaker or lower-layer | The test is relevant but insufficient | Strengthen or add a bridge/higher layer |
| Evidence has no required chapter | It may be decorative or reveal an omitted requirement | Investigate before keeping or removing |
| Required and observed propositions conflict | Test, code, or specification disagrees | Report and resolve the contradiction |
| Local propositions match but handoffs differ | The suite proves disconnected lookalike paths | Add a bridge or consume real output |
| Numerical or performance evidence lacks activation | The run may exercise the wrong path | Connect it to production activation |

An unmatched bottom-up proposition is not automatically useless. It may reveal an incidental
behavior change omitted from the PR description. If so, revise the effective intention and add a
preservation or boundary chapter.

## Use upstream as evidence and precedent

The model applies to upstream testcases without modification. Upstream tests enter the bottom-up
tunnel like any other executable evidence: extract their literal propositions, identify their
evidence boundaries, and decide how they relate to the downstream change.

Do not treat "upstream has a test" as a conclusion. An upstream testcase may play several
different roles:

| Upstream role | Meaning for downstream review |
| --- | --- |
| Inherited evidence | The exact upstream test runs against the changed downstream runtime and its proposition still applies |
| Preservation baseline | Expected downstream behavior should remain equal to the upstream implementation |
| Reusable testcase | The upstream test can be extended or parameterized to cover the downstream case |
| Design precedent | Its fixture, oracle, or granularity is useful, but it is not executable evidence for this change |
| Scope contrast | It covers CUDA, CPU, or another topology and makes the missing NPU-specific layer explicit |
| Negative precedent | Literal review exposes that the upstream test itself is weaker than its name or assumed role |

Always record the exact upstream revision. A current upstream testcase is not necessarily the test
present in the version pinned by the downstream repository.

### Upstream review reminder

The eventual review skill can include this compact output section without changing the core mental
model:

```text
Upstream reference
  revision:
  testcase or artifact:
  literal proposition:
  role: inherited / baseline / reusable / precedent / contrast / negative precedent
  downstream delta:
  executed by downstream CI: yes / no / unknown
  remaining proof gap:
```

Search upstream before proposing new evidence, especially for shared model components,
parallelism, checkpointing, compilation, and configuration behavior. Prefer extending or reusing a
stable upstream test when its fixture and oracle represent the same contract.

Do not copy upstream tests mechanically. A downstream patch can change the activation path,
backend, representation, distributed topology, or numerical contract. In that case the upstream
test may be a template or baseline rather than sufficient evidence.

### Two upstream examples

At upstream TorchTitan revision `96276d86577cf3e3bd29de72586e76af62010a55`,
`tests/unit_tests/test_activation_checkpoint.py::TestApplyAC::test_correctness` is a useful compact
precedent. One testcase constructs several activation-checkpoint variants and compares forward
outputs, input gradients, and parameter gradients with an uncheckpointed reference. Those
observations jointly establish one proposition: the selected recomputation policies preserve
forward and backward semantics. The model should not generate a separate testcase for every
output, gradient, parameter, or policy row.

The same file separates recomputation selection, FLOPs, CUDA memory, and correctness. These are
separate stories or chapters because they have different terminal claims, oracles, and required
execution layers.

By contrast, at the same revision,
`tests/unit_tests/test_compile_moe.py::TestApplyCompile::test_grouped_mm_compiles_and_runs` applies
compile to one model, constructs a separate `GroupedExperts` module, executes that module, and
asserts only its output shape. Its literal proposition is a successful grouped-expert forward with
the expected shape in that setup. The assertions do not by themselves establish that the executed
module was compiled or that compiled numerics match eager. Upstream status does not exempt a test
from literal-proposition and handoff review.

## Evidence minimality and stopping rule

The model should generate the smallest connected evidence set that closes the required story
bundle, not one testcase per proposition, changed function, branch, or runtime seam.

One testcase may be preferable when several observations are causally inseparable parts of one
terminal proposition. For example, forward output, input gradients, and parameter gradients can
belong in one semantic-equivalence test. Split them only when they require different scenarios,
oracles, layers, or produce meaningfully independent failures.

Add a narrow seam test only when it contributes at least one unique value:

- rejects a fault not rejected by broader evidence;
- protects a changed or historically fragile handoff;
- covers a distinct boundary, default, invalid, or compatibility partition;
- runs at a required evidence layer absent elsewhere;
- provides substantial diagnostic localization at low maintenance cost;
- preserves a stable public or upstream contract directly.

Before proposing a testcase, ask:

> If this testcase is removed, which named required claim, plausible fault, scenario partition,
> execution layer, or handoff becomes unprotected?

If there is no concrete answer, do not add the testcase.

Also check for dominated evidence. Testcase A dominates testcase B when A establishes B's relevant
proposition over at least the same domain and layer, while B adds no unique fault sensitivity,
boundary, upstream contract, speed, or diagnostic value. Retain B only when that additional value
is explicit.

Stop generating tests when:

1. every mandatory terminal claim has adequate evidence;
2. changed or risky handoffs are continuous;
3. required default, preservation, and boundary partitions are covered;
4. required execution layers are represented;
5. every retained test adds a named, unique proof or diagnostic contribution;
6. removing any proposed additional test would create no identified proof gap.

This is an evidence-minimization rule, not a weakening of the proof model. The target is sufficient
discrimination with low suite cost, not maximal testcase count.

## Individual testcase information inside a narrative

Narrative organization must not erase testcase-level auditability. Preserve a compact evidence
card for each retained testcase:

| Field | Purpose |
| --- | --- |
| Literal proposition | What setup, operation, and assertions establish |
| Story role | Which terminal claim and chapter it advances |
| Scenario and source | Inputs and who owns their meaning |
| Activation and observation | How the path is selected and what is inspected |
| Oracle | Independent reason the expected result is correct |
| Fault rejected | Concrete wrong implementation that should fail |
| Handoff | Earlier fact consumed and later fact enabled |
| Evidence boundary | What this layer and scenario do not prove |
| Result | Exact revision, command, status, and artifact when applicable |

Order cards by causality. Each card should say which earlier fact it consumes, which proposition it
establishes, and which later step can rely on it. Branch cards should say where they leave the main
path and why.

## Readability and identifiers

Do not narrate using positional labels such as `M3`, `M3a`, or `M7b`. They encode document position
rather than meaning and become unstable when stories are split or reordered.

Use semantic story and step titles in prose:

```text
Configuration selects Muon compilation.
Real construction consumes the selection and installs the compiled callable.
The disabled branch remains eager.
```

When a traceability system needs keys, use stable semantic keys such as:

```text
activation/config-selection
activation/runtime-use
activation/disabled-default
equivalence/eager-preservation
equivalence/compiled-equivalence
equivalence/training-equivalence
performance/acceptance
```

The keys exist for machinery. Readers should not need them to understand the story. A testcase
function name already identifies the testcase and normally does not need another artificial ID.

## Muon compilation example

The feature idea is to optionally compile the Muon Newton-Schulz calculation while preserving
optimizer behavior.

### Activation story

```text
compile configuration
  -> derive enable state and backend
  -> real Muon construction installs the compiled Newton-Schulz callable
  -> LMO executes that callable

disabled configuration
  -> no torch.compile call
  -> LMO remains eager
```

Terminal claim:

> User configuration controls which Newton-Schulz implementation the real optimizer executes.

### Equivalence story

```text
refactored eager LMO preserves legacy 2D and 3D composition
  -> activated compiled calculation matches eager over supported shapes
  -> deterministic training loss and grad_norm remain equivalent

compile disabled
  -> full expert update remains equal to the parent under the relevant dtype contrast
```

Terminal claim:

> Introducing the replaceable and compiled calculation does not change supported optimizer or
> training results.

### Performance story

```text
production activation path
  -> compiled workload after warmup
  -> declared throughput, latency, and memory measurements
  -> comparison with a named baseline and acceptance threshold
```

### What an existing helper test literally proves

Suppose a test executes three cases against `get_muon_compile_options()`:

- global compilation disabled with Muon selected returns disabled and no backend;
- compilation enabled without Muon selected returns disabled and no backend;
- compilation enabled with Muon selected returns enabled and the configured backend.

Its literal proposition is limited to those three helper outputs. It does not prove that:

- the real optimizer builder consumes the tuple;
- `torch.compile` is called;
- the correct function or backend is compiled;
- the returned callable is used by LMO;
- numerical behavior remains unchanged.

The helper test is clear, useful supporting evidence. A name claiming that runtime compilation "is
selected" overstates it. A more literal name is:

```python
test_get_muon_compile_options_requires_enable_and_muon_component
```

### Missing activation evidence

A component-level test should build through the real Muon configuration path while replacing
`torch.compile` with a recorder that returns a sentinel callable. The test should verify that:

- the recorder receives the Newton-Schulz function, selected backend, `fullgraph=True`, and
  `dynamic=True`;
- the constructed optimizer's LMO invokes the returned callable;
- the component result matches the eager calculation.

Those observations support one causal proposition: configured compilation becomes the callable
used for the optimizer calculation. Actual accelerator compilation, dynamic-shape reuse,
numerical stability, and performance remain separate evidence.

## Draft mode

```text
derive required proposition
  -> select scenario, oracle, fault, and execution layer
  -> implement testcase
  -> re-extract its literal proposition
  -> compare literal and required propositions
  -> revise until adequate
  -> connect it into the evidence story
```

Re-extraction is mandatory. It detects a strong intended test whose implemented assertions prove
only a helper detail.

## Review mode

```text
extract literal proposition from existing testcase
  -> infer broader contract separately, if justified
  -> identify required PR proposition
  -> compare them through the proposition gate
  -> retain, narrow, strengthen, supplement, split, or remove
  -> connect accepted evidence into the story bundle
```

Drafting and review therefore share one core model. They differ only in which end supplies the
first artifact.

## Completion

The two tunnels meet when:

1. every mandatory top-down chapter has adequate bottom-up evidence;
2. every retained testcase maps to a named claim, risk, or preservation contract;
3. every changed runtime handoff is continuous across the evidence stories;
4. independent stories are connected wherever one assumes activation or output established by
   another;
5. partial matches, unsupported scope, and merge-only evidence remain explicit;
6. a hostile implementation cannot pass the evidence bundle while violating the effective PR
   intention.

The final review artifact is not merely a generated test list. It contains the required story
graph, the observed evidence graph, and a visible reconciliation between them.

## Open design questions

- When may a checked interface fixture replace a real producer-to-consumer bridge?
- Which multi-rank and accelerator claims require permanent CI rather than merge-time evidence?
- How should automated extraction represent uncertainty in a literal proposition?
- When should a single causally focused testcase be preferred over several narrow seam tests?
- How can the model suggest hostile mutations without overfitting tests to implementation detail?
- Which story and evidence fields belong in human-facing PR prose versus machine-readable review
  output?
