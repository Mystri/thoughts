# TorchTitan-NPU Test Design Policy and Skill -- Progress Draft

Last updated: 2026-08-08

Status: discussion draft. Nothing has been committed or pushed.

## 1. Workspace and objective

- Repository: `/mnt/workspace/gitCode/cann/torchtitan-npu/torchtitan-npu-master`
- Base: `cann/torchtitan-npu` `master` at `8c5fb12`
- Draft branch: `testcase-review-policy`
- Comparison branch: `override-refactor` at `f18da25`

Objective: define a repository policy and an executable agent skill that can design, generate, and review testcase evidence close to upstream TorchTitan standards.

The system must answer two different questions:

1. Does the existing test suite protect the current version's functional contracts?
2. Does a proposed change add enough evidence for the contract delta introduced by the change?

Testcase count and line coverage cannot answer either question by themselves.

## 2. What is already settled

### 2.1 Tests are not classified by one exclusive label

`bugfix`, `new model`, `activation checkpointing`, and `parallelism` are not alternatives at the same level.

A change can be described as:

```text
Intent:             bugfix
Product scope:      existing model behavior
Primary owner:      MoE parallelism
Secondary areas:   TP, DTensor, backward gradients
Semantic impact:   gradient correctness
Evidence layers:   CPU/DTensor contract + multi-rank numerical regression
```

Only the primary owner is singular, and only because a permanent testcase needs a stable home. Secondary areas and evidence layers are sets.

### 2.2 Permanent tests are organized by stable ownership

- CPU UTs live with stable subsystems such as config, checkpoint, activation checkpointing, converter, model, optimizer, or distributed logic.
- Accelerator integration cases can combine several capabilities, models, and parallelisms.
- Bug and PR lineage is preserved by Git history and the PR evidence table, not by per-PR test directories.

### 2.3 Coverage has two meanings

| Meaning | Question | Calculator support |
| --- | --- | --- |
| Structural coverage | Did execution touch this line or branch? | Yes |
| Functional/contract coverage | Would the test reject an incorrect implementation of the promised behavior? | No |

Structural coverage may help confirm activation. It is not proof of behavior.

### 2.4 Existing tests may satisfy a new PR

A PR does not need a new test file for every production edit. It may cite an existing testcase only when it demonstrates:

- the testcase is collected and executed by CI;
- the changed path is activated;
- its assertion represents the affected contract;
- its oracle is independent;
- reverting or mutating the change makes it fail.

### 2.5 Change-story evidence remains useful

For each changed behavior:

```text
Before -> Change -> Activation -> After -> Oracle -> Sensitivity -> Protected scope
```

This describes the delta introduced by a PR. It does not replace the baseline functional contract inventory accumulated in permanent tests.

## 3. What upstream TorchTitan does

Upstream does not publish a strict, exclusive taxonomy. Its practice uses several dimensions implicitly.

### 3.1 Filesystem axis: ownership and execution layer

CPU files are organized by subsystem, for example:

```text
test_activation_checkpoint.py
test_checkpoint.py
test_config_manager.py
test_dataloader.py
test_expert_parallel.py
test_quantization.py
```

Integration suites are broader:

```text
features.py
models.py
flux.py
h100.py
```

GraphTrainer is a separate ownership boundary and has local tests for numerics, determinism, graph passes, CPU offload, profiling, and performance passes.

Reference: <https://github.com/pytorch/torchtitan/blob/main/tests/README.md>

### 3.2 Proof axis: semantic impact

Upstream requires:

- identical loss for non-computation changes;
- representative convergence for computation changes;
- GPU integration tests for parallelism;
- throughput/memory/profile evidence for performance claims.

Reference: <https://github.com/pytorch/torchtitan/blob/main/CONTRIBUTING.md>

### 3.3 CPU coverage collection is not a semantic gate

Upstream CPU CI runs the full unit suite with `pytest-cov` and writes a coverage XML. No visible `fail-under` threshold or percentage-based merge gate is configured in that workflow.

Reference: <https://github.com/pytorch/torchtitan/blob/main/.github/workflows/unit_test_cpu.yaml>

Semantic confidence instead comes from behavior-oriented assertions, accumulated regression tests, full-suite execution, code-owner review, and PR-specific numerical/integration evidence.

### 3.4 Example: activation-checkpointing refactor

The refactor crossed activation checkpointing, config, trainer, model parallelization, GraphTrainer, RL, and integration configs. Upstream did not choose only one category. It:

- updated the owning activation-checkpointing unit tests;
- updated affected consumers and integration definitions;
- proved bitwise-identical loss and `grad_norm` because computation was not intended to change.

Reference: <https://github.com/pytorch/torchtitan/pull/3674>

### 3.5 Example: Qwen3.5 model landing

The model landing crossed architecture, data, RoPE, state dict, TP, CP, EP, PP, and training integration. Evidence included HF numerical parity, sharding checks, and integration configuration rather than one unit test per source function.

Reference: <https://github.com/pytorch/torchtitan/commit/fd712e814b581f2c1cdc01743d4a53ce1c01d0b4>

### 3.6 Example: MoE TP gradient bugfix

The PR clearly documented a TP-without-EP backward-gradient failure and supplied before/after gradient evidence across TP1/TP2/TP4. It proposed, but apparently did not land, a permanent DTensor regression test.

Reference: <https://github.com/pytorch/torchtitan/pull/4054>

This is evidence that upstream practice is strong but not complete. Our policy should preserve the PR proof as a permanent sensitive testcase when feasible.

## 4. Baseline coverage and change-delta coverage

These must be designed separately.

### 4.1 Baseline functional contract map

This is independent of the current PR and is organized by contract owner.

| Contract | Scenarios | Oracle | Permanent testcase | CI layer |
| --- | --- | --- | --- | --- |
| AC preserves results | no AC vs selective/full AC | uncheckpointed execution | output/input-grad/param-grad comparison | CPU |
| AC changes recomputation | each policy and forced FQN | analytical op counts | recomputed-op/FLOP test | CPU |
| EP permutation | rank/expert layouts and empty groups | manually derived permutation | normal and boundary layouts | CPU |
| Checkpoint restores state | save, mutate, load | original state | round-trip test | CPU/NPU as needed |

This map answers: what functionality of the current version is protected?

### 4.2 Change-delta evidence map

This is generated for a commit or PR.

| Change claim | Affected baseline contracts | Before proof | After proof | Permanent evidence |
| --- | --- | --- | --- | --- |
| Fix TP-without-EP gradients | gradient placement and reduction | parent under-reduces | candidate matches reference | DTensor/TP regression |

This map answers: what did this PR add, remove, correct, or promise to preserve?

## 5. Proposed core model of a testcase

The earlier rules about naming, placement, deterministic seeds, mocks, boundaries, and CI are important practices. They are not the intrinsic core of a testcase.

The proposed semantic core is four elements. It must be separated from the mechanics used to implement an executable testcase.

### Core 1: Contract

A precise proposition the testcase intends to prove.

```text
For inputs/configurations in domain D, behavior B must satisfy relation R.
```

Examples:

- Selective AC changes recomputation policy but preserves outputs and gradients.
- EP permutation converts rank-major token order to expert-major order.
- Enabling converter X replaces the declared target modules and leaves excluded modules unchanged.

Without a contract, a test has no reason to exist beyond exercising code.

### Core 2: Scenario and domain

A meaningful partition of the contract's input/configuration/state space.

The scenario defines the conditions under which the claim is expected to hold or fail: normal case, boundary, invalid input, disabled/default path, topology, dtype, shape, checkpoint version, or another contract-relevant partition.

Without a scenario/domain, “test the module” is too broad to determine either adequate inputs or expected behavior.

### Core 3: Oracle

An independent reason the expected observation is correct.

Possible oracles:

- manually derived expected values;
- an unoptimized/reference PyTorch implementation;
- an established implementation such as upstream or Hugging Face;
- a mathematical invariant;
- before/after equivalence for a non-computation refactor;
- round-trip or metamorphic relations.

Candidate output copied into a golden file is not an independent oracle.

### Core 4: Fault sensitivity

The plausible incorrect implementation or regression that the testcase must reject.

Sensitivity can be demonstrated by:

- parent-fail/candidate-pass for a bugfix;
- reverting the relevant hunk;
- injecting a representative mutation;
- showing that a wrong placement, omitted registration, disabled override, wrong scale, missing backward, or corrupted state causes the assertion to fail.

A testcase that cannot name any plausible wrong behavior it distinguishes is probably weak even if it executes many lines.

### Executable realization

The semantic design becomes executable through:

- preconditions and fixtures;
- stimulus;
- activation evidence;
- observation points;
- assertions against the oracle;
- cleanup, determinism, parameterization, and CI selection.

Activation is mandatory evidence whenever a selector, patch, converter, override, config, fallback, compile mode, or distributed topology could leave the target path inactive. For a directly called pure function, the call itself may be sufficient activation evidence.

These mechanics implement the semantic core but are not independent design categories.

The compact model is:

```text
Test design = Contract + Scenario/domain + Oracle + Fault model

Executable testcase = Setup + Stimulus/activation + Observation/assertion
```

## 6. Proposed large axes for testcase design

The four cores describe one testcase. The following axes organize a test plan or suite.

### Axis A: Contract lifecycle / change delta

- Established contract baseline
- Bugfix: intended contract already exists but implementation violates it
- New capability: adds a new contract
- New model/training engine: adds a profile of standard contracts plus model-specific contracts
- Refactor: contract set should remain unchanged
- Performance change: functional contracts remain, resource/performance contract changes
- Intentional breaking migration: old contract is replaced with explicit compatibility behavior

This resolves the false choice between `bugfix` and `new model`: they describe different contract deltas.

### Axis B: Capability ownership

Possible owners include:

- config and registry
- data/tokenizer/dataloader
- model architecture and shared model components
- operator/kernel/converter/override/patch
- activation checkpointing
- optimizer
- checkpoint/state dict
- compile/GraphTrainer
- distributed parallelism: DP/FSDP, TP, CP, EP, PP
- observability/profiling/tooling
- CI/test infrastructure

One primary owner determines placement. All affected secondary owners remain attached to the claim.

### Axis C: Semantic property

- selection/activation/control flow
- forward value/shape/dtype
- backward/gradient/autograd
- placement/collective/distributed state
- persistence/checkpoint compatibility
- determinism/numerical equivalence/convergence
- memory/performance
- error/warning/failure behavior

This axis determines what must be observed and asserted.

### Axis D: Evidence execution layer

- CPU unit/contract test
- NPU component/kernel test
- NPU multi-rank/distributed test
- numerical regression or convergence run
- end-to-end training/config test
- checkpoint compatibility test
- performance/memory benchmark

The required evidence is the union of applicable contracts and risks, not a Cartesian product of every tag.

## 7. Proposed overlap algorithm

For each independent change claim:

1. Identify the contract delta from Axis A.
2. Select a primary contract owner and all secondary affected owners from Axis B.
3. Select every semantic property changed or promised unchanged from Axis C.
4. Find existing permanent tests for those contracts.
5. Mark each contract `PROVEN`, `PARTIAL`, or `UNPROVEN`; do not produce a coverage percentage.
6. Draw the runtime evidence chain for the changed behavior and identify every changed handoff between producers and consumers.
7. Add the smallest set of testcases whose four cores are complete.
8. For every changed handoff, require either a producer-derived fixture or a thin cross-seam testcase that feeds the producer's real output to the consumer. A separately invented lookalike fixture is not handoff evidence.
9. Choose the cheapest execution layer that can prove each claim; escalate when hardware, distributed, numerical, checkpoint, or performance behavior cannot be represented below it.
10. Add an interaction testcase only when the overlap creates distinct behavior at the intersection.

One testcase may prove several cells when its activation, oracle, and assertions explicitly cover them.

## 8. Draft skill workflow derived from the core

```text
Identify contract inventory
  -> classify contract delta
  -> map primary and secondary owners
  -> identify changed semantic properties
  -> define plausible fault models
  -> inspect existing test sensitivity
  -> design missing Contract/Scenario/Oracle/Fault-model evidence
  -> select CPU/NPU/numerical/E2E/performance layer
  -> verify collection and execution
  -> report PROVEN/PARTIAL/UNPROVEN
```

This is stronger than both code coverage and a generic checklist because it explains why each assertion discriminates correct from incorrect behavior.

## 9. Case study: PR 439 and handoff continuity

PR 439, `[fix] disable greedy packing for SDPA to ensure correct per-doc attn masking`, deliberately told a three-stage story:

```text
recipe selects SDPA + block_causal
  -> Decoder constructs a dense document mask
  -> Trainer supplies the mask
  -> ScaledDotProductAttention forwards it to functional SDPA
```

Reference: <https://gitcode.com/cann/torchtitan-npu/pull/439>

### 9.1 Why the original evidence looked convincing

The tests were layered and diagnostic:

- the real Qwen3 recipe was checked for SDPA + `block_causal` on every layer;
- a hand-written matrix independently checked causal isolation for two artificial EOS-delimited documents;
- `Decoder.get_attention_masks` was checked to return a dense mask;
- Trainer post-dataloading was checked to place the mask in `extra_kwargs`;
- functional SDPA was spied to prove that it received the mask with `is_causal=False`;
- separate dataloader tests proved that block-causal SDPA retained greedy packing while ordinary causal SDPA disabled it;
- the ten targeted CPU tests passed against the exact pinned upstream TorchTitan commit.

This evidence was fault-sensitive for the local implementation seams. Removing configuration allowance, corrupting the tested EOS mask, omitting Trainer plumbing, dropping `attn_mask`, or restoring `is_causal=True` would fail a distinct testcase.

### 9.2 What the story did not prove

The mask tests used invented token batches in which EOS tokens were placed at document joins. The dataloader test separately proved that real samples were packed, but it did not feed that packed batch into mask construction.

Using PR 439's own `ChatDataLoader`, tokenizer, dataset records, and recipe produces document starts at `positions` indices 0, 68, and 136. The token immediately before joins 68 and 136 is not EOS. PR 439's EOS-derived mask therefore reports:

```text
position_resets [0, 68, 136]
boundary 68  cross_doc_allowed True
boundary 136 cross_doc_allowed True
```

The implementation and its unit tests were internally consistent about EOS, but the assumption was not the real packed-chat boundary contract. Chat templates can also emit EOS inside one conversation, creating the opposite error: a false document split.

The later fix, commit `c5170ce` / PR 481, changed the source of truth to dataloader `positions == 0` and added `test_position_boundaries_ignore_message_eos_and_split_packed_samples`. That test covers both adversarial facts at once:

- an internal message EOS is not a packed-document boundary;
- a true packed-sample join is a boundary even when no EOS is present in shifted model inputs.

### 9.3 Policy lesson

The four-part semantic core for an individual testcase remains useful. PR 439 does not show that it needs a fifth element. It shows that a multi-test change story needs an additional suite-level property:

```text
Handoff continuity:
For each changed producer -> consumer seam, the consumer evidence must use the
producer's real output, or a fixture derived from the producer's explicit
contract and checked against it.
```

For this feature, the stronger evidence chain is:

```text
real ChatDataLoader batch (tokens + positions)
  -> boundary source of truth
  -> dense block-causal mask semantics
  -> Trainer extra_kwargs
  -> Decoder/layer/inner-attention forwarding
  -> functional SDPA attn_mask with is_causal=False
  -> no cross-document influence invariant
```

Not every arrow requires a large end-to-end test. Small seam tests remain preferable for diagnosis, but at least one thin test must bridge each changed handoff. This case also sharpens `Scenario/domain`: fixtures must include domain facts that can falsify the implementation's assumed representation, not only examples that agree with it.

## 10. Current repository draft artifacts

| Artifact | Responsibility |
| --- | --- |
| `.agents/rules/testing.md` | Normative test-evidence policy; needs revision after the core is settled |
| `.agents/skills/torchtitan-npu-test-review/SKILL.md` | Design/generate/review workflow; currently valid but not integrated |
| `references/evidence-matrix.md` | Change-class to evidence-layer mapping |
| `references/testcase-examples.md` | Valid and invalid examples |
| `references/review-output.md` | PASS/NEEDS EVIDENCE/BLOCK output contract |

Planned integrations after the core draft is accepted:

1. `.agents/AGENTS.md`
2. `.agents/README.md`
3. `torchtitan-npu-code-reviewer`
4. `docs/test_guides/test_design.md`
5. GitCode PR template
6. `premerge-accuracy-check` exact-vs-tolerance rules

## 11. Next points to settle

1. Are Contract, Scenario/domain, Oracle, and Fault model the correct semantic core, or is one missing/redundant?
2. Is the separation between semantic design and executable realization correct, with activation mandatory only when path selection can be ambiguous?
3. Should fault sensitivity be mandatory for every testcase, or mandatory at the change-claim level so several tests can jointly provide it?
4. Should baseline contracts be maintained as a checked-in catalog, derived by the skill from tests/docs/code, or use a small checked-in catalog only for high-risk subsystems?
5. Is semantic property the right third planning axis, or should forward/backward/distributed/checkpoint/numerical be expressed as contract subtypes instead?
6. When one testcase proves several overlapping contracts, what explicit evidence must the PR provide to avoid accidental overclaiming?
7. Should handoff continuity be mandatory for every changed producer/consumer seam, or only for seams whose representation or source of truth changes?
