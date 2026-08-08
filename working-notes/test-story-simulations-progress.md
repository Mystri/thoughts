# TorchTitan-NPU Test-Story Simulations -- Progress Report

Last updated: 2026-08-08

Status: discussion and simulation draft. No policy/skill integration, commit, or push has been
performed as part of this report.

## 1. Executive answer

Yes: the current principles can generate test stories with the layered clarity of PR 439.

The simulations also show how to make those stories stronger than PR 439. A good story is not
merely a list of unit tests for changed functions. It is a connected evidence graph:

```text
producer contract
  -> representation at the handoff
  -> transformation contract
  -> runtime plumbing
  -> consumer contract
  -> externally meaningful behavior
```

Each changed node needs a contract, scenario, oracle, and fault model. Each changed edge needs
handoff continuity: either the consumer test uses the producer's real output, or the fixture is
derived from and checked against the producer's explicit contract.

The four simulated reviews were:

| Commit | Change shape | Why selected |
| --- | --- | --- |
| `c5170ce` | Bugfix across data, mask construction, Trainer, SDPA/Varlen | Direct follow-up to PR 439 and a strong connected story |
| `8c5fb12` | Checkpoint/state-dict bugfix | Tests aliases and metadata, but durable file-level round-trip evidence is incomplete |
| `d063011` | Optimizer compile feature | Demonstrates selection tests that do not activate the claimed feature |
| `ac1c6d3` | Distributed FSDP precision feature | Strong CPU contracts, but real multi-rank and model integration remain separate evidence |

The simulations support making the **test story the central PR-review artifact**, while keeping
four proof obligations for every testcase used by that story:

```text
Contract + Scenario/domain + Oracle + Fault model
```

The resulting hierarchy is:

```text
PR intention/specification
  -> test story/evidence argument
  -> testcase proof steps
  -> executable observations and artifacts
```

The simulations add three suite/change-level requirements:

1. Runtime-chain decomposition.
2. Handoff continuity between changed producers and consumers.
3. Separation of permanent baseline evidence from one-time PR/manual evidence.

## 2. Record of the discussion so far

### 2.1 Test organization

We rejected a single exclusive classification such as "bugfix versus model versus
parallelism." Tests are organized along several axes:

- lifecycle/change intent: bugfix, feature, refactor, performance, migration;
- primary and secondary capability owners;
- semantic property: selection, values, gradients, placement, persistence, numerics,
  performance, failure behavior;
- execution layer: CPU, NPU component, multi-rank, numerical, E2E, checkpoint, performance.

Permanent tests live with stable contract owners. PR/bug lineage remains in history and the PR
evidence table rather than per-PR test directories.

### 2.2 Coverage

We separated structural coverage from functional coverage:

- structural coverage asks whether execution touched a line or branch;
- functional coverage asks whether a testcase rejects a plausible incorrect implementation of
  the promised behavior.

A coverage calculator helps with the first question, not the second.

### 2.3 Semantic core

The current individual-test model is:

```text
Test design = Contract + Scenario/domain + Oracle + Fault model

Executable testcase = Setup + Stimulus/activation + Observation/assertion
```

Activation is required when configuration, registration, patching, converter selection,
fallback, compilation, or topology could leave the target path inactive.

### 2.4 PR 439 lesson

PR 439 was exemplary in local structure:

```text
config -> mask construction -> Trainer forwarding -> functional SDPA
```

Its original tests independently protected those seams. However, the mask fixtures assumed EOS
tokens occurred at packed-document joins. The real dataloader represented document joins using
`positions == 0`, and shifted model inputs could omit EOS at the join. Chat templates could also
emit EOS inside one conversation.

PR 439 therefore proved an internally consistent EOS-based story rather than the real
packed-chat boundary contract. This led to the handoff-continuity principle.

Detailed analysis is recorded in:

`progress/testcase-review-policy/DRAFT.md`, section "Case study: PR 439 and handoff continuity."

### 2.5 Action in this report

The current action was to apply the emerging method to other representative commits, without
changing their code, and ask:

1. Can the change be expressed as a readable PR-439-style story?
2. Which existing tests prove each link?
3. Are adjacent links connected, or do they use unrelated lookalike fixtures?
4. Would parent/revert/mutation behavior make the tests fail?
5. Which claims remain CPU-only, manual-only, or unproven?
6. What test story should have been requested during review?

## 3. Current draft artifacts

### 3.1 Repository drafts

Working repository:

`/mnt/workspace/gitCode/cann/torchtitan-npu/torchtitan-npu-master`

Branch and base:

```text
branch: testcase-review-policy
base:   master at 8c5fb12
```

Untracked drafts:

- `.agents/rules/testing.md`
- `.agents/skills/torchtitan-npu-test-review/`

These have not yet been integrated into `.agents/AGENTS.md`, committed, or pushed.

### 3.2 Progress documents

- `progress/testcase-review-policy/DRAFT.md`: core model, axes, upstream comparison, and PR 439
  case study.
- `progress/test-story-simulations/PROGRESS.md`: this report and the four simulated reviews.

## 4. Simulation method

For each commit:

1. Read the commit description and diff against its parent.
2. Classify intent, owners, semantic properties, and risk amplifiers.
3. Convert each behavior claim into a runtime chain.
4. Inventory tests added or modified by the commit.
5. Map tests to claims using activation, oracle, sensitivity, and protected scope.
6. Mark each claim `PROVEN`, `PARTIAL`, or `UNPROVEN` instead of assigning a percentage.
7. Perform a simulated review using repository severity rules.
8. Generate the smallest connected test story that would close the gaps.

The simulations distinguish:

- production correctness findings;
- missing merge-time evidence;
- missing permanent baseline protection;
- optional diagnostic improvements.

## 5. Reusable test-story form

The test story should appear after the basic PR information and before the raw test-result list.
The basic information establishes what the author intends; the story explains why the submitted
evidence proves it.

Recommended order:

```text
1. PR description and classification
2. Intention ledger
3. Test story graph
4. Narrative testcase cards
5. Intention-to-evidence matrix
6. Commands, results, and artifacts
7. Gaps, limitations, and review conclusion
```

For each change claim, record:

| Field | Question |
| --- | --- |
| Contract | What proposition must hold? |
| Source of truth | Which producer owns the meaning of the input/state? |
| Scenario/domain | Which normal, adversarial, boundary, disabled, and invalid cases matter? |
| Activation | What proves the changed path was selected? |
| Oracle | Why is the expected observation independently correct? |
| Fault model | Which plausible wrong implementation must fail? |
| Handoff | Is the consumer using the producer's real output or verified contract fixture? |
| Evidence layer | Can CPU prove it, or is NPU/multi-rank/E2E required? |
| Preservation | Which default and unaffected paths must remain unchanged? |

A PR-439-style narrative can then be generated as:

```text
selector/config
  -> producer/source-of-truth representation
  -> transformation or state transition
  -> runtime dispatch/plumbing
  -> consumer API/kernel/collective
  -> behavioral or persistence result
```

### 5.1 Narrative testcase card

Each testcase in the story needs more than a path and name. Give it a compact narrative card in
the PR description or generated review report. Do not duplicate the whole card as a long test
docstring.

| Field | Description |
| --- | --- |
| Story role | Why this testcase exists and which step of the story it proves |
| Intention claim | Stable claim ID from the PR intention ledger |
| Given/source | Preconditions and who owns the meaning of the fixture |
| When/activation | Stimulus and evidence that the changed path was selected |
| Then/observation | Externally relevant state, value, call, placement, or file observed |
| Oracle | Independent reason the expected observation is correct |
| Fault rejected | Concrete parent behavior or mutation that the assertion kills |
| Handoff | Previous producer contract consumed and next consumer contract enabled |
| Evidence boundary | What CPU/mock/single-rank evidence does not prove |
| Result | Exact command, commit, status, and artifact where applicable |

"Narrative" does not mean replacing testcase details with one suite-level paragraph. Preserve
the individual card for auditability, but order and connect the cards by causality. Each card
should state which earlier fact it consumes, which proposition it establishes, and which later
step can rely on it. Branch cards should say where they leave the main path and why.

The description should normally take two to five sentences. Its purpose is to let a reviewer
understand the proof contribution without reading fixture plumbing first.

Example:

```text
T2 -- test_position_boundaries_ignore_message_eos_and_split_packed_samples

Role: proves the semantic center of packed-document isolation (I2). Given token values that
deliberately contradict document boundaries, and position resets supplied by the dataloader
contract from T1, Decoder must keep internal EOS positions in the same document and close the
mask at resets without EOS. The oracle is the explicit visibility relation implied by the reset
indices. It rejects the old EOS-derived implementation. This is CPU mask evidence; it does not
prove that an NPU attention kernel consumes the mask.
```

### 5.2 The story is a graph even when written as prose

The PR may narrate the happy path linearly, but the evidence model may branch:

```text
                         -> disabled/default preservation
config -> producer -> transform -> consumer -> intended outcome
                         -> invalid/unsupported behavior
                         -> multi-rank or checkpoint compatibility
```

Do not force overlapping changes into one artificial sequence. Use one main spine plus branches
for preservation, boundaries, errors, topologies, and non-functional claims.

## 6. Simulation A -- `c5170ce`: packing positions as document boundaries

### 6.1 Classification

```text
Intent:             bugfix
Primary owner:      attention-mask construction
Secondary owners:  ChatDataLoader, Trainer, SDPA, Varlen metadata, DeepSeek-V4 dispatch
Semantic impact:   document isolation and runtime plumbing
Risk factors:      packed data, multiple attention backends, CP branch, silent numerical error
```

### 6.2 Contract delta

Before:

- document identity was inferred from EOS token values;
- internal ChatML EOS could create false splits;
- actual packed joins without EOS could be missed.

After:

- dataloader position resets are the source of truth;
- dense SDPA masks and Varlen metadata use the same boundaries;
- Trainer builds masks before downstream forwarding/parallel preparation;
- causal SDPA still disables packing, while supported block-causal paths retain it.

### 6.3 Runtime story

```text
ChatDataLoader emits tokens + per-document positions
  -> positions == 0 identifies document starts
  -> Decoder builds dense mask or VarlenMetadata
  -> Trainer places positions and attention_masks in extra_kwargs
  -> SDPA receives dense mask, or NPU Varlen receives metadata
  -> documents cannot attend across packed boundaries
```

### 6.4 Existing evidence assessment

| Claim | Existing evidence | Assessment |
| --- | --- | --- |
| Internal EOS is not a document boundary | Adversarial mask assertions | `PROVEN` on CPU |
| Join without EOS is a boundary | Position reset assertions | `PROVEN` on CPU |
| Dataloader produces resets at real joins | Real `ChatDataLoader` packing testcase | `PROVEN` on CPU |
| Dense SDPA mask is forwarded with `is_causal=False` | Functional SDPA spy | `PROVEN` on CPU |
| Trainer preserves positions and forwards mask | Trainer post-dataloading testcase | `PROVEN` on CPU |
| Varlen metadata has correct cumulative boundaries | Multi-batch explicit metadata oracle | `PROVEN` on CPU |
| DeepSeek-V4 packing selection does not treat integer layers as configs | Flat-config tests | `PROVEN` on CPU |
| CP behavior for dense masks/metadata | No real CP case in the commit | `UNPROVEN` or outside declared scope |
| Real NPU training consumes the corrected boundary data | No new numerical/E2E artifact in the commit | `PARTIAL` |

### 6.5 Why this is a strong story

The adversarial fixture represents both sides of the original fault:

- EOS exists where there is no document boundary;
- a document boundary exists where there is no EOS.

The dataloader test independently confirms the representation contract, and the mask tests use
that checked representation. This satisfies handoff continuity without requiring one large
end-to-end unit test.

The Varlen testcase adds a multi-batch scenario, which rejects implementations that accidentally
flatten only one batch or compute incorrect cumulative sequence offsets.

### 6.6 Simulated review conclusion

The core CPU bugfix story is convincing and substantially stronger than the original PR 439
story.

One scope decision remains mandatory: either declare block-causal SDPA plus CP unsupported and
reject it clearly, or add real CP sharding/collective evidence. The production path explicitly
contains CP handling, so silence is not sufficient evidence.

An NPU smoke/numerical case should also demonstrate that the corrected metadata reaches the
selected NPU attention path. The CPU tests prove representation and dispatch semantics, not HCCL
or NPU-kernel behavior.

### 6.7 Generated test story

Intentions for this story:

- `A1`: the dataloader's position resets are the document-boundary source of truth;
- `A2`: dense and Varlen masks isolate those documents while preserving causal visibility;
- `A3`: Trainer and attention consumers receive the derived representation;
- `A4`: default causal/pretraining behavior is preserved;
- `A5`: CP and NPU support is either proven or explicitly rejected.

- **T-A1 -- `test_packed_chat_positions_mark_every_sample_join` (`A1`, CPU).** Role: establish
  the producer contract before testing any mask. Given the real `ChatDataLoader`, sample
  processor, tokenizer, and packed records, observe exact position-reset indices in its output.
  Those indices are the oracle because the dataloader owns packing. This rejects a producer that
  concatenates samples without resetting positions and supplies the fixture contract consumed by
  T-A2/T-A3; it does not prove attention isolation.

- **T-A2 -- `test_position_boundaries_ignore_message_eos_and_split_packed_samples` (`A1/A2`,
  CPU).** Role: prove the semantic center. Given internal EOS tokens and true joins without EOS,
  build the dense mask from position resets and assert visibility within a document and exclusion
  across resets. The explicit reset-derived visibility relation is the oracle. It kills the old
  EOS implementation and hands the mask contract to T-A4/T-A5; it does not prove runtime
  forwarding.

- **T-A3 -- `test_position_resets_create_varlen_metadata_for_packed_samples` (`A1/A2`, CPU).**
  Role: prove the alternative Varlen representation uses the same source of truth. Given two
  batches with different reset layouts, assert exact cumulative sequence offsets, maximum
  lengths, CPU device, and integer dtype. Manually derived offsets are the oracle. This rejects
  batch flattening and off-by-one boundaries; it does not exercise the NPU Varlen kernel.

- **T-A4 -- `test_trainer_forwards_position_derived_mask_in_extra_kwargs` (`A3`, CPU).** Role:
  prove the Trainer handoff. Feed the checked positions/mask scenario through the real patched
  post-dataloading method and observe that positions and masks are placed in `extra_kwargs`, not
  stranded in `extra_inputs`. The Decoder result from T-A2 is the handoff oracle. This rejects a
  correct builder that is never delivered to model/PP stages.

- **T-A5 -- `test_sdpa_forwards_position_mask_and_disables_builtin_causal_mask` (`A3`, CPU).**
  Role: prove the final CPU API handoff. Call the real SDPA wrapper with T-A2's mask, spy the
  functional SDPA boundary, and assert the exact mask plus `is_causal=False`. The PyTorch API
  contract is the oracle. This rejects dropped/inverted masks and double causal masking; it does
  not prove the NPU implementation consumes the argument correctly.

- **T-A6 -- causal SDPA and pretraining preservation cases (`A4`, CPU).** Role: protect behavior
  outside the opt-in feature. Real configs must keep causal SDPA un-packed and leave pretraining
  packing unchanged. Existing behavior is the oracle. These cases reject an overly broad
  dataloader patch and form the preservation branch of the story.

- **T-A7 -- `test_block_causal_sdpa_with_cp_is_rejected_or_sharded_correctly` (`A5`,
  multi-rank).** Role: close the declared topology boundary. If unsupported, construction must
  raise a user-facing error before training; if supported, a real CP mesh must shard positions and
  masks consistently. The declared support contract and an unsharded reference are the oracles.

- **T-A8 -- packed-document isolation integration (`A3/A5`, NPU).** Role: close the externally
  meaningful behavior. Change document A while holding document B fixed and verify that document
  B outputs or loss contributions remain invariant under the selected NPU attention path. The
  non-interference relation is the oracle. This catches a kernel or distributed path that accepts
  the mask argument but ignores its semantics.

## 7. Simulation B -- `8c5fb12`: DeepSeek HF checkpoint export

### 7.1 Classification

```text
Intent:             bugfix
Primary owner:      checkpoint/HF export
Secondary owners:  DeepSeek-V3.2 state-dict adapter, DCP metadata, safetensors consolidation
Semantic impact:   persistence, distributed export, load compatibility
Risk factors:      partial rank state, aliases, EP/PP/FSDP, file headers, old/no-MTP path
```

### 7.2 Contract delta

The commit fixes two distinct claims:

1. Consolidation mapping must be filtered by global DCP `state_dict_metadata`, not by one rank's
   local state dict and not by the unfiltered original HF index.
2. DeepSeek-V3.2 MTP aliases for embedding, norm, and LM head must be exported for every MTP
   layer, then removed during import so the main-model weights remain canonical.

### 7.3 Runtime story

```text
local model state on each rank
  -> DeepSeek adapter emits canonical HF keys + available MTP aliases
  -> DCP writer gathers global metadata
  -> consolidation mapping drops keys absent globally
  -> safetensors/index contain valid dtype, shape, and shard entries
  -> HF reader + adapter restore canonical model keys and values
```

### 7.4 Existing evidence assessment

| Claim | Existing evidence | Assessment |
| --- | --- | --- |
| All MTP layers receive three shared aliases | Exact key and identity assertions | `PROVEN` on CPU |
| Partial state dict emits only available aliases | Embedding-only testcase | `PROVEN` on CPU |
| Import removes aliases and restores canonical keys | Adapter round-trip | `PROVEN` on CPU |
| No-MTP behavior remains unchanged | Disabled testcase | `PROVEN` on CPU |
| Filter uses global metadata and preserves remote-rank keys | DCP metadata mock distinguishes local/global sets | `PROVEN` at seam level |
| Adapter mapping is not mutated | Explicit source-mapping assertion | `PROVEN` on CPU |
| Produced safetensors header is valid | Reported manual single-process validation | `PARTIAL`, not permanent |
| EP/PP/FSDP partial ranks consolidate correctly | Reported 2-rank checks | `PARTIAL`, not permanent |
| Exported checkpoint reloads and resumes DeepSeek-V3.2 MTP | No model-specific automated save/load smoke | `UNPROVEN` permanently |

### 7.5 What is convincing

The metadata testcase has a good fault-discriminating fixture:

- `kept.weight` is local and global;
- `remote_expert.weight` is absent locally but present globally;
- `unused_mtp.weight` is absent globally.

This rejects both wrong filters: filtering by the local state would drop a remote expert, while
using the original mapping would retain an invalid key.

The adapter tests also protect partial-rank behavior rather than requiring every alias source to
be locally available.

### 7.6 Remaining handoff gap

The adapter and metadata/consolidation logic are tested separately with mocks. No permanent test
feeds adapter output through a real DCP/HF writer and then opens the resulting safetensors/index.

The PR description reports exactly that manual validation, including two-rank partial and DTensor
cases. This may be sufficient merge-time evidence when artifacts are reviewable, but it does not
become a durable baseline regression test.

### 7.7 Simulated review conclusion

The local unit story is good and the reported manual evidence materially reduces merge risk. The
permanent suite remains `PARTIAL` for the central user-visible result: a valid, reloadable
DeepSeek-V3.2 MTP HF checkpoint under distributed partial state.

This is an example where "tests passed" and "PR proof was adequate" can both be true while the
repository baseline still deserves a follow-up test.

### 7.8 Generated test story

Intentions for this story:

- `B1`: every enabled MTP layer receives aliases for available shared weights;
- `B2`: global DCP metadata, not a local rank or stale HF mapping, determines exported keys;
- `B3`: consolidation produces structurally valid safetensors and index files;
- `B4`: export and reload preserve canonical model values under partial/distributed state;
- `B5`: no-MTP and unavailable-local-weight behavior remains valid.

- **T-B1 -- `test_to_hf_exports_available_shared_aliases_for_every_mtp_layer` (`B1/B5`, CPU).**
  Role: establish the adapter's key-expansion contract. Given canonical embedding, norm, and LM
  head values plus an MTP layer count, assert every enabled layer receives the correct alias and
  that partial local state emits only aliases whose source exists. The model/HF naming contract
  is the oracle. This rejects missing aliases and unsafe assumptions that every rank owns every
  shared value; it produces the state consumed by the writer story.

- **T-B2 -- `test_dcp_filters_hf_mapping_using_global_metadata` (`B2`, CPU).** Role: prove the
  global-membership decision. Use three disjoint categories--local/global, remote/global, and
  absent/global--then assert consolidation keeps the first two, drops the third with a warning,
  and does not mutate the adapter mapping. The deliberately constructed set relation is the
  oracle. This rejects both local-only and unfiltered mappings; it does not write a real file.

- **T-B3 -- `test_deepseek_v32_mtp_export_writes_valid_safetensors_headers` (`B1/B2/B3`, CPU or
  component).** Role: bridge adapter and metadata logic into the real writer. Feed T-B1-style
  state and T-B2-style mapping through DCP/HF export in a temporary directory, then use
  `safe_open` and the index to verify every listed tensor has a readable dtype, shape, shard, and
  value. The safetensors format reader is the independent oracle. This catches mocks that agree
  while consolidation still writes an invalid header.

- **T-B4 -- `test_deepseek_v32_mtp_hf_round_trip_restores_canonical_values` (`B3/B4`,
  checkpoint).** Role: prove the user-visible persistence result. Save a known canonical state,
  reload it through the HF reader and adapter, and compare key set, dtype, shape, and values. The
  original in-memory state is the oracle. This rejects exports that are syntactically valid but
  semantically unloadable or duplicate aliases on import.

- **T-B5 -- `test_two_rank_partial_state_consolidates_remote_and_mtp_aliases` (`B2/B4`,
  multi-rank).** Role: prove the distributed handoff represented only by mocks in T-B2. Give each
  rank a different local subset, consolidate from real global DCP metadata, reload, and compare
  with the union reference. This rejects rank-local filtering and missing DTensor/EP shards. Use
  CPU Gloo when the contract is device-independent; add NPU coverage if HCCL/storage behavior is
  involved.

- **T-B6 -- `test_no_mtp_export_keeps_existing_hf_key_set` (`B5`, CPU).** Role: protect the
  established path. With `num_mtp_modules=0`, export the canonical state and assert no MTP alias
  appears while normal keys remain unchanged. The parent key set is the oracle. This catches a
  global adapter change that leaks feature-specific aliases into all checkpoints.

## 8. Simulation C -- `d063011`: compile Muon Newton-Schulz

### 8.1 Classification

```text
Intent:             feature, with an incidental computation change
Primary owner:      Muon optimizer
Secondary owners:  compile config, optimizer dispatcher, swap optimizer, training numerics
Semantic impact:   selection, compilation, optimizer values, performance
Risk factors:      compile activation, dynamic shapes, default path, distributed optimizer,
                   unsupported converter combination
```

### 8.2 Claimed runtime story

```text
CompileConfig(enable=True, components contains "muon")
  -> optimizer dispatcher forwards compile config
  -> Muon container selects backend
  -> DistributedMuon calls torch.compile with fullgraph + dynamic
  -> LMO invokes compiled Newton-Schulz
  -> optimizer updates match eager numerics while reducing compile overhead
```

### 8.3 Existing evidence assessment

| Claim | Existing evidence | Assessment |
| --- | --- | --- |
| Config selection matrix enables only the `muon` component | Helper parameterization | `PROVEN` locally |
| Refactored instance LMO matches the old eager composition | Fixed-seed 2D reference | `PROVEN` for that case |
| `torch.compile` is called with the selected backend | No testcase activates construction | `UNPROVEN` permanently |
| `fullgraph=True` and `dynamic=True` are effective | Documentation only | `UNPROVEN` |
| Dispatcher and swap paths deliver compile config | No end-to-end construction assertion | `UNPROVEN` |
| Compiled 2D/3D and multiple shapes match eager | Reported 100-step training comparison only | `PARTIAL` |
| Compile-disabled optimizer is unchanged from parent | Not tested against parent | `UNPROVEN` |
| `npu_smla` conflict fails clearly | Documentation says unsupported; no guard/test | `UNPROVEN` |
| Performance improves without numerical regression | One reported environment/curve | `PARTIAL` review evidence |

### 8.4 Simulated correctness finding

`DistributedMuon.step_experts()` changed from:

```python
self.lmo(g, ...)
```

to:

```python
self.lmo(g.to(dtype=self.communication_dtype), ...)
```

This cast is unconditional, including when Muon compile is disabled. When `g.dtype` differs from
`communication_dtype`, the default eager optimizer update changes numerically. The existing
`test_muon_lmo_matches_legacy_2d` bypasses `step_experts()` and therefore cannot detect it.

The reported eager-versus-compile comparison appears to compare two candidate configurations.
It does not prove parent-versus-candidate equivalence for the compile-disabled path.

Simulated finding:

```text
[S1] Compile-disabled expert updates are not protected from the unconditional dtype cast.
```

The minimum resolution is one of:

- remove the unrelated cast;
- restrict it to an explicitly justified compile requirement and prove eager/default
  preservation;
- declare it as a computation change and supply an independent component oracle plus
  representative convergence evidence.

### 8.5 Simulated evidence finding

The only new compile testcase checks a pure option parser. It would still pass if:

- `torch.compile` were never called;
- the wrong function were compiled;
- `fullgraph` or `dynamic` were omitted;
- the compiled callable were never used by LMO;
- dispatcher or swap plumbing dropped the configuration.

Simulated finding:

```text
[S2] The permanent suite never activates the feature named by the PR.
```

### 8.6 Generated test story

The narrative has seven intention claims:

```text
C1 selection:     compile is selected only when globally enabled for the Muon component
C2 construction:  the declared Newton-Schulz target and compile options are constructed
C3 use:           the compiled callable is the callable used by the optimizer
C4 plumbing:      ordinary and swap optimizer builders preserve the selection
C5 equivalence:   compiled updates agree with eager updates over the supported shape domain
C6 preservation:  disabled/default Muon behavior remains equal to the parent behavior
C7 boundaries:    unsupported combinations fail clearly; supported NPU behavior is numerical
                  and operationally sound
```

The testcase cards form one story: select the mode, construct and use the callable, deliver the
configuration through every builder, prove the computation, then protect the default and
unsupported branches.

- **T-C1 -- `test_muon_compile_selection_requires_enable_and_component` (`C1`, CPU).** Role:
  establish the story's selector. Parameterize global enablement and component membership, and
  assert compile is selected only when both conditions hold. The configuration truth table is
  the oracle. This rejects always-on, global-only, and component-only selection; it does not show
  that any optimizer consumes the result.

- **T-C2 -- `test_distributed_muon_compiles_newton_schulz_with_declared_options` (`C2/C3`,
  CPU).** Role: turn the selector into an activated construction claim. Spy on `torch.compile`,
  assert the exact Newton-Schulz target, backend, `fullgraph=True`, and `dynamic=True`, return a
  sentinel callable, and drive LMO far enough to prove that sentinel is invoked. The spy and
  sentinel are complementary oracles: one proves construction and the other proves use. This
  rejects a correct option parser whose result is ignored at runtime.

- **T-C3 -- `test_compile_disabled_uses_original_newton_schulz_without_compiling` (`C1/C6`,
  CPU).** Role: close the disabled branch immediately after the construction step. Build a real
  Muon container with compile disabled, assert `torch.compile` is not called, and assert the eager
  callable remains selected and used. The parent's eager callable is the oracle. This catches
  accidental eager compilation and feature leakage into the default path.

- **T-C4 -- `test_optimizer_dispatcher_forwards_muon_compile_config` (`C4`, CPU).** Role: prove
  the primary configuration handoff. Construct through the real optimizer config and dispatcher,
  then observe the selected callable on the resulting Muon instance rather than calling the
  selection helper directly. T-C1 defines the expected selection and T-C2 defines the expected
  constructed state. This rejects a dispatcher that drops, rewrites, or applies the configuration
  to the wrong optimizer.

- **T-C5 -- `test_swap_muon_forwards_muon_compile_config` (`C4`, CPU).** Role: prove the second
  builder changed by the commit. Enter through the real swap-optimizer path and observe the same
  selected callable and compile construction contract as T-C4. The ordinary builder's resulting
  Muon state is the reference contract. This rejects a feature that works in the main dispatcher
  but silently disappears when optimizer swapping is enabled.

- **T-C6 -- `test_compiled_newton_schulz_matches_eager_for_2d_3d_and_dynamic_shapes`
  (`C3/C5`, CPU component plus NPU component).** Role: prove the mathematical chapter of the
  story. Run fixed inputs through independent eager and compiled callables for representative 2D,
  3D, and shape-changing sequences, then compare outputs and optimizer updates within the
  declared tolerance. The eager implementation is the numerical oracle; a counting backend on
  CPU proves durable graph activation, while the NPU backend case proves actual device support.
  This rejects a compile call that exists but is bypassed, specialized to one shape, or changes
  the Newton-Schulz result.

- **T-C7 -- `test_compile_disabled_expert_step_matches_parent_update` (`C6`, CPU).** Role:
  protect the baseline after the new feature has been proven. Use an FP32 expert gradient with
  BF16 communication dtype and compare the disabled candidate update with a parent/reference
  implementation. The parent update, not candidate eager-versus-compiled agreement, is the
  oracle. This specific dtype contrast kills the unconditional-cast mutation found in the
  simulated review.

- **T-C8 -- `test_muon_compile_rejects_unsupported_npu_smla_combination` (`C7`, CPU config or
  NPU component).** Role: make the stated boundary executable. Request the incompatible
  combination and assert a precise user-facing error before training begins. The documented
  support contract is the oracle. If the combination is actually supported, replace this card
  with an activated integration case; documentation alone cannot decide the behavior.

- **T-C9 -- deterministic NPU numerical and performance run (`C5/C7`, merge evidence).** Role:
  complete the user-visible story at its necessary execution layer. Compare eager and compiled
  runs from the same checkpoint, seed, topology, data, and optimizer settings; inspect full-
  precision loss and `grad_norm`, compile/recompile count, and performance after warmup. The eager
  run and the repository's deterministic numerical contract are the oracles. This evidence can
  substantiate backend numerics and performance for the reviewed environment, but it is not a
  substitute for the permanent activation and preservation tests above.

## 9. Simulation D -- `ac1c6d3`: parameter-level FSDP precision

### 9.1 Classification

```text
Intent:             distributed feature with performance claim
Primary owner:      FSDP mixed-precision policy
Secondary owners:  config, DeepSeek-V4 patterns, parallelize ordering, PyTorch private patch,
                   gradients, TorchAO wrappers
Semantic impact:   all-gather/compute dtype, gradient dtype, collective behavior, performance
Risk factors:      PyTorch internal signatures, TP/EP transforms, shared parameters, wrappers,
                   multi-rank collectives, default path
```

### 9.2 Runtime story

```text
model-owned FQN patterns
  -> final post-converter/post-TP parameter names are matched
  -> selected ordinary parameters receive a preservation marker
  -> FSDPParam captures the marker
  -> selected parameters bypass unit-level param_dtype cast
  -> one mixed-dtype all-gather feeds computation
  -> mixed gradients are normalized to FP32
  -> native reduce-scatter remains the collective path
```

### 9.3 Existing evidence assessment

| Claim | Existing evidence | Assessment |
| --- | --- | --- |
| `*` and `**` match intended FQN semantics | Explicit nested fixtures | `PROVEN` on CPU |
| Invalid/unmatched patterns fail or warn | Error/warning cases | `PROVEN` on CPU |
| Shared aliases are deduplicated by parameter identity | Alias fixture | `PROVEN` on CPU |
| TorchAO extension parameters are rejected/excluded | Wrapper-marker fixtures | `PROVEN` on CPU |
| Selected/unselected params become FP32/BF16 in one FSDP unit | Real single-rank `fully_shard` test | `PROVEN` on CPU/single rank |
| Both gradient types become FP32 | Real backward plus reduce wrapper tests | `PROVEN` locally |
| Native path remains for no preserved parameter | Wrapper delegation test | `PROVEN` locally |
| Upstream private signatures are guarded | Runtime signature validation | `PROVEN` by import/activation |
| DeepSeek-V4 configs own independent pattern lists | Exact config assertions | `PROVEN` on CPU |
| Patterns run after converters/TP/EP and before first `fully_shard` | Production ordering inspected, no activated order test | `PARTIAL` |
| One all-gather and one reduce-scatter remain under mixed dtype | Single rank/mocked collective cannot prove | `UNPROVEN` permanently |
| Multi-rank gradients/placements are correct | Reported 4P run, no permanent multi-rank testcase | `PARTIAL` |
| 4% performance gain and unchanged peak memory | Reported measurement only | `PARTIAL` |

### 9.4 What is especially strong

This is the strongest of the simulated feature unit suites. It does more than test a helper:

- a real FSDP unit is constructed;
- selected and unselected parameter dtypes are observed after forward;
- backward gradients are observed;
- failure contracts for reduce dtype and wrappers are explicit;
- config list aliasing is checked;
- the patch fails fast if upstream private signatures change.

This is close to a PR-439-style story at the CPU contract layer.

### 9.5 Missing evidence layer

The feature's defining distributed claims include collective count, mixed-dtype packing, and
multi-rank reduction. A one-rank mesh and a mocked `foreach_reduce` cannot prove them.

Simulated finding:

```text
[S2] Real multi-rank collective and gradient semantics are not permanently tested.
```

The PR's 4P numerical/performance evidence is valuable merge-time proof, but the test policy
should request a stable multi-rank scenario for the smallest representative unit.

### 9.6 Generated test story

The narrative has eight intention claims:

```text
D1 selection:     model-owned patterns select the intended final ordinary parameters
D2 ordering:      selection occurs after name-changing transforms and before FSDP sharding
D3 forward:       selected parameters preserve FP32 while unselected parameters use unit policy
D4 backward:      mixed gradients are normalized correctly
D5 collectives:   the feature retains one valid all-gather and reduce-scatter path
D6 preservation:  unselected/default units remain on native FSDP behavior
D7 boundaries:    aliases, wrappers, invalid patterns, and representative TP/EP names are handled
D8 outcome:       multi-rank results match an unsharded reference with acceptable NPU numerics,
                  memory, and performance
```

The testcase cards follow parameter identity from the model namespace into FSDP, through forward
and backward, and finally through real multi-rank collectives. Default and boundary branches are
attached where they diverge from that main path.

- **T-D1 -- `test_fsdp_precision_patterns_match_final_parameter_fqns` (`D1/D7`, CPU).** Role:
  establish which parameters enter the story. Exercise `*` and `**`, invalid and unmatched
  patterns, shared aliases, and TorchAO wrapper parameters, then compare selected parameter
  identities with a fixture-defined expected set. Parameter identity and the declared pattern
  grammar are the oracles. This rejects spelling-only matches, double marking through aliases,
  silent unmatched configuration, and unsupported wrapper selection; it does not prove when the
  selector runs in model construction.

- **T-D2 -- `test_deepseek_v4_parallelize_marks_after_converters_before_fully_shard`
  (`D1/D2`, CPU).** Role: bridge pattern selection into the real parallelization sequence. Use a
  representative converter/name change, record final names presented to the selector, and assert
  marking completes before the first `fully_shard` call. The production ordering and T-D1's
  identity set define the oracle. This rejects correct matching logic invoked on stale names or
  after FSDP has already captured parameter state.

- **T-D3 -- `test_single_fsdp_unit_uses_bf16_and_preserved_fp32_parameters` (`D3/D4`, CPU
  component).** Role: prove that the marker changes real FSDP behavior. Construct one actual FSDP
  unit containing selected and unselected parameters, run forward and backward, and observe FP32
  for selected computation plus the declared unit dtype for unselected computation and normalized
  gradient dtypes. The explicit mixed-precision policy and an unsharded calculation provide the
  oracle. This rejects tests that stop at marker attributes while FSDP ignores them.

- **T-D4 -- `test_unselected_fsdp_unit_follows_native_mixed_precision` (`D6`, CPU component).**
  Role: protect the branch with no selected parameters. Construct the same unit without matching
  markers and compare its parameter, forward, and gradient behavior with native FSDP mixed
  precision. Native FSDP is the oracle. This rejects a global patch that changes every unit even
  when the model has not requested parameter preservation.

- **T-D5 -- `test_two_rank_mixed_dtype_fsdp_matches_unsharded_reference` (`D3/D4/D8`,
  multi-rank CPU or NPU).** Role: extend the component result across an actual sharding boundary.
  On both ranks compare forward output, input gradients, parameter gradients, dtypes, and
  placements with an unsharded reference using the same initial state and inputs. The unsharded
  model is the independent mathematical oracle. This rejects single-rank behavior that fails when
  parameters or gradients are distributed.

- **T-D6 -- `test_mixed_dtype_fsdp_uses_one_all_gather_and_one_reduce_scatter` (`D5`,
  multi-rank).** Role: prove the collective-efficiency claim without replacing the computation
  under test. Instrument real collective entry points, run T-D5's forward/backward operation, and
  assert the expected all-gather and reduce-scatter counts while retaining its numerical checks.
  The native one-unit FSDP collective schedule is the oracle. This rejects an implementation that
  is numerically correct only because it performs extra per-dtype collectives.

- **T-D7 -- `test_ep_tp_transformed_names_match_model_owned_patterns` (`D1/D2/D7`, multi-rank
  component).** Role: cover the highest-risk secondary owners named by the change. Apply the
  representative TP/EP transforms used by the target model, assert final parameter identities
  selected by the model-owned patterns, then prove those identities reach the FSDP marker step.
  Final transformed names plus parameter identity are the oracles. This rejects patterns that
  work on the untransformed model but miss or misidentify parameters after parallelization; it
  intentionally avoids a topology Cartesian product.

- **T-D8 -- deterministic NPU numerical and performance run (`D5/D8`, merge evidence).** Role:
  complete the story in the environment where the feature's precision and performance claims
  matter. Compare the intended policy with its reference using deterministic full-precision loss
  and `grad_norm`, at least 10 measured steps after warmup, throughput, peak memory, and run-to-
  run variation. The reference policy and repository numerical rules are the oracles. This proves
  the reviewed topology and environment, while T-D5 and T-D6 remain the durable protection for
  multi-rank semantics and collective count.

## 10. Cross-simulation comparison

| Commit | Local contract tests | Handoff continuity | Default path | Required hardware layer | Simulated result |
| --- | --- | --- | --- | --- | --- |
| `c5170ce` | Strong | Strong for data->mask->Trainer | Causal/packing paths covered | NPU/CP scope remains | Core story convincing; scope evidence needed |
| `8c5fb12` | Strong aliases and metadata | Missing real writer/file bridge | No-MTP covered | Distributed save/load desirable | Merge evidence plausible; permanent baseline partial |
| `d063011` | Weak for the named feature | Config->compile->LMO not connected | Unconditional cast not protected | NPU compile/numerics required | Would request changes |
| `ac1c6d3` | Very strong CPU suite | Model-order handoff partial | Native reduce path covered | Real multi-rank NPU required | CPU design convincing; distributed proof incomplete |

The method differentiates these commits without using testcase count or line coverage. In
particular:

- 48 Muon tests do not compensate for failing to activate the new compile feature.
- 17 FSDP precision tests provide meaningful CPU confidence but cannot prove a multi-rank
  collective claim.
- a small three-set checkpoint fixture is highly convincing because it discriminates local,
  global, and invalid mapping behavior.
- one adversarial attention fixture is more valuable than many ordinary EOS-delimited examples.

## 11. Principles refined by the simulations

### 11.1 A test story is a connected graph, not a checklist

PR 439's style should be retained: small tests at named seams make failures diagnosable. The
reviewer must additionally inspect whether adjacent tests agree on the representation and source
of truth.

### 11.2 The source of truth belongs in the contract

"Build the correct mask" is incomplete. A stronger contract says:

> Build document isolation from the dataloader's position-reset boundaries, independent of token
> values.

Likewise:

- checkpoint membership comes from global DCP metadata;
- compile activation comes from the actual constructed callable, not option parsing;
- distributed precision comes from actual FSDP parameters/collectives, not only marker state.

### 11.3 Preservation evidence must compare the right baseline

Comparing eager and compile modes within the candidate does not prove that the candidate's eager
path matches the parent. Every incidental change outside the feature selector needs its own claim
and parent/mutation evidence.

### 11.4 Unsupported combinations are negative contracts

If a feature is documented as incompatible with CP, a converter, wrapper tensor, checkpoint
version, or optimizer mode, the implementation should either:

- reject the combination with a user-facing error and testcase; or
- support it and provide the corresponding evidence.

A documentation sentence alone does not protect training from a silent or obscure failure.

### 11.5 Manual PR proof and permanent tests have different jobs

Manual multi-rank, loss, checkpoint, and performance artifacts can be sufficient to review a
specific environment. They should be recorded as merge-time evidence, but must not be counted as
permanent baseline coverage.

The review output should therefore report both:

```text
Merge evidence:     sufficient / partial / missing
Permanent baseline: protected / partial / unprotected
```

### 11.6 A behavior invariant can close the final story

API spies are good activation and handoff evidence. A final invariant makes the story less tied
to implementation details:

- changing document A does not change document B outputs;
- save then load preserves canonical checkpoint values;
- compiled and eager optimizer updates agree;
- distributed and unsharded reference gradients agree.

## 12. Proposed generated-review output

For future simulated or real reviews, produce:

### A. Basic PR information

- problem and user-visible motivation;
- change type and affected owners;
- supported scope and explicit exclusions;
- computation, persistence, distributed, or performance impact;
- parent/base and candidate/head revisions.

### B. Intention ledger

Assign stable IDs to independently provable intentions:

| ID | Intention claim | Source | Kind | Mandatory? |
| --- | --- | --- | --- | --- |
| I1 | Enabling the recipe selects block-causal SDPA on every layer | PR + config diff | new behavior | yes |
| I2 | Packed samples cannot attend across position-reset boundaries | bug report + data contract | corrected behavior | yes |
| I3 | Ordinary causal SDPA remains un-packed | existing contract | preservation | yes |

Sources are not limited to the PR prose. Also inspect the issue/reproducer, user documentation,
diff, changed callsites, upstream contracts, and behavior changes that the author did not mention.

### C. Change-story graph

```text
<producer> -> <representation> -> <transform> -> <plumbing> -> <consumer> -> <behavior>
```

Add branches for default behavior, invalid inputs, unsupported combinations, compatibility,
multi-rank behavior, numerics, and performance when they are part of the intention ledger.

### D. Narrative testcase cards

Describe each testcase using story role, claim ID, Given/source, activation, observation, oracle,
fault rejected, handoff, evidence boundary, and actual result.

### E. Intention-to-evidence matrix

| Intention | Story step | Test/evidence | Oracle | Fault rejected | Layer | Status |
| --- | --- | --- | --- | --- | --- | --- |

Every mandatory intention must end as `PROVEN`, `PARTIAL`, or `UNPROVEN`; do not average the
statuses into a percentage.

### F. Findings

Only concrete correctness or evidence gaps, ordered by severity.

### G. Generated testcase plan

The smallest connected set of tests, with proposed names and clear ownership.

### H. Two conclusions

```text
Merge-time evidence conclusion
Permanent baseline conclusion
```

## 13. Validation performed during this action

Repository rules and skills read:

- `.agents/AGENTS.md`
- `.agents/skills/torchtitan-npu-code-reviewer/SKILL.md`
- `.agents/rules/testing.md`
- applicable config, distributed, models, and patches rules
- system `skill-creator` guidance for forward-testing an evolving skill

Diffs inspected:

```text
c5170ce^..c5170ce
8c5fb12^..8c5fb12
d063011^..d063011
ac1c6d3^..ac1c6d3
```

CPU validation was run against pinned upstream TorchTitan commit
`ac13e536c84e7f6647b14fa9375c3c8a8a2b8578`:

```text
100 collected
100 passed
20 warnings
elapsed: 1.25s pytest time
```

Files included:

- attention SDPA, TND, and chat packing tests;
- DeepSeek-V3.2 adapter and checkpoint patch tests;
- full Muon optimizer unit file;
- FSDP parameter precision tests;
- DeepSeek-V4 precision pattern config test.

An initial run used the wrong local upstream checkout and failed during import because that
checkout's config API did not match the plugin's pin. It was rerun with the exact pinned upstream
commit and passed. This is environment evidence, not a product finding.

Not executed:

- NPU component tests;
- multi-rank HCCL tests;
- training numerical comparisons;
- checkpoint export/reload subprocess smokes;
- performance measurements.

The simulated review does not claim those layers passed.

## 14. Draft decisions and next discussion points

### Recommended decisions

1. Adopt the test story as the central PR-review artifact: intention -> evidence argument ->
   testcase proof steps.
2. Preserve each testcase as a narrative evidence card with its story role, claim ID, scenario,
   activation, observation, oracle, rejected fault, handoff, evidence boundary, and result.
3. Treat activation, oracle independence, fault sensitivity, and handoff continuity as proof
   validity conditions, not optional descriptive metadata.
4. Require each changed handoff to use real producer output or a producer-checked contract
   fixture.
5. Require separate reporting for merge evidence and permanent baseline coverage.
6. Require parent preservation for every change outside the declared selector, even if two
   candidate modes agree with each other.
7. Treat unsupported combinations as explicit negative contracts.

### Points still worth grilling

1. Should every changed representation seam require a thin bridge test, or may an explicitly
   versioned/stable interface contract replace it?
2. When is manual multi-rank evidence sufficient for merge, and when must a permanent smoke test
   land in the same PR?
3. Should a feature review be blocked when the feature is activated only by manual numerical
   evidence but has no CPU activation test?
4. Which high-risk baseline contracts deserve a checked-in catalog rather than being derived by
   the review skill each time?
5. Should the skill automatically propose parent/mutation commands, or only describe the expected
   failure when executing them is too expensive?

## 15. Next implementation action after discussion

After these conclusions are accepted or revised:

1. update `.agents/rules/testing.md` with the intention ledger, central story, narrative testcase
   card, proof gate, handoff continuity, and two evidence conclusions;
2. update `torchtitan-npu-test-review/SKILL.md` to generate the intention ledger, story graph,
   ordered testcase cards, and intention-to-evidence matrix;
3. teach generation to propose the smallest connected set of testcase cards, rather than an
   unstructured list of tests or a category quota;
4. add the four simulations as concise valid/invalid examples in the skill references;
5. validate the skill;
6. perform a fresh forward-test on a commit not used to derive the rules;
7. integrate into repository agent guidance, commit, and push only after review.

## 16. Discussion update -- should the test story be the core?

### 16.1 The idea in its strongest form

The test story should be the core **unit of PR evidence and review**.

It is the artifact that answers:

> Why does this collection of individual tests, numerical runs, checkpoint artifacts, and
> performance measurements prove the PR's intention?

Coverage, ownership, semantic property, oracle type, fault sensitivity, and evidence layer are
different projections of that story. They help reviewers inspect it from different angles, but
none should become a separate checklist disconnected from the central argument.

This is a better organizing idea than beginning with a matrix of test categories. A category
matrix can tell us where evidence lives; the story tells us why the evidence matters.

### 16.2 Where the idea needs a guardrail

A story can be coherent, readable, and wrong. PR 439 demonstrated this: every local step agreed
with the EOS assumption, so the narrative felt complete even though the assumption did not match
the dataloader contract.

Therefore, not every "slice" has the same logical status:

- **ownership, location, and execution layer** are descriptive views;
- **coverage/status** is a conclusion about the relationship between intention and evidence;
- **oracle, activation, fault sensitivity, and handoff continuity** are validity conditions for
  accepting the story as proof.

An oracle is not optional metadata. Without an independent oracle, the story may only show that
several implementation components agree with each other. Fault sensitivity asks whether the
story distinguishes the candidate from a plausible wrong implementation. Handoff continuity
asks whether adjacent proof steps refer to the same real representation.

The refined statement is:

```text
The test story is the core review artifact.
Testcases are proof steps in the story.
Oracles, activation, sensitivity, and handoff continuity validate those proof steps.
Ownership, coverage views, and evidence layers help organize and inspect the story.
```

For an individual testcase, the smallest useful semantic unit remains a proposition with a
scenario, oracle, and fault model. Calling the whole narrative the core of one singular testcase
would blur the boundary between one proof step and the complete PR argument.

### 16.3 Three separate objects

To avoid circular reasoning, keep these objects separate:

1. **PR intention/specification** -- what must be true if the PR succeeds.
2. **Test story/evidence argument** -- why the evidence is sufficient to establish that intention.
3. **Testcases and artifacts** -- observations used as premises in the argument.

The PR description is evidence about intention, but not the only authority. The diff may contain
an unmentioned behavior change; existing public contracts may require preservation; a bug report
may define a source of truth more precisely than the implementation.

### 16.4 How to decide whether the story covers PR intention

#### Step 1: derive the effective intention

Build the intention ledger from all relevant sources:

- user-visible problem or issue reproducer;
- promised after behavior in the PR description;
- enable/default/disabled and invalid behavior;
- supported models, shapes, dtypes, topologies, checkpoint versions, and backends;
- behavior explicitly promised unchanged;
- numerical, performance, memory, or compatibility claims;
- implicit behavior changes discovered in the diff and affected callsites;
- upstream or repository contracts that the change overrides.

If the prose and diff disagree, do not silently redefine the intention around the implementation.
Report the mismatch and make the author resolve it.

#### Step 2: decompose intention into atomic claims

Each claim must be independently provable or explicitly out of scope. Include four kinds:

1. desired new/corrected behavior;
2. activation and runtime plumbing;
3. preservation/default/compatibility behavior;
4. boundaries, invalid inputs, unsupported combinations, numerics, and non-functional claims.

#### Step 3: establish bidirectional traceability

Check both directions:

- every mandatory intention claim maps to one or more story steps and evidence items;
- every testcase maps back to an intention, a preservation contract, or a named risk.

The first direction detects missing proof. The second detects decorative tests that add volume but
do not strengthen the PR argument.

#### Step 4: verify vertical and horizontal completeness

**Vertical completeness** follows the runtime path from selector to externally meaningful result:

```text
selection -> producer -> representation -> transform -> plumbing -> consumer -> outcome
```

**Horizontal completeness** covers the applicable domain partitions:

```text
enabled/default/disabled
normal/adversarial/boundary/invalid
model variants and changed callsites
dtype/shape/topology/checkpoint version
eager/compile or save/load
```

Do not create a Cartesian product. Select partitions that correspond to plausible faults and
declared support.

#### Step 5: verify evidence adequacy

A claim is not proven merely because it has a mapped testcase. The evidence must have:

- activation of the claimed path;
- an independent oracle;
- sensitivity to parent/revert or a representative mutation;
- continuity across changed handoffs;
- the correct execution layer;
- explicit limitations and protected scope.

For intention claim `I`, use the following logical gate:

```text
PROVEN(I) =
  path activated
  AND observation represents I
  AND oracle is independent
  AND plausible wrong behavior fails
  AND changed handoffs are continuous
  AND evidence layer can observe the claimed behavior
  AND required preservation/boundary cases pass
```

Overall intention coverage is achieved only when every mandatory claim is `PROVEN`. `PARTIAL`
and `UNPROVEN` claims remain visible; they are not averaged into a percentage.

#### Step 6: perform the hostile counterexample review

Ask:

> Can I implement something that passes every testcase in this story but still violates the PR
> intention?

Try at least these mutations:

- selector/config is correct but the runtime ignores it;
- producer and consumer use different meanings for the same fixture;
- transformation uses the wrong source of truth;
- plumbing forwards a mask/state object that the real consumer never receives;
- enabled mode works but the default path regresses;
- single-rank works but collective placement/reduction is wrong;
- adapter/mocks agree but the produced checkpoint file is invalid;
- numerical modes agree with each other after both diverged from the parent;
- unsupported combinations fail late or silently.

If such a counterexample exists, either add evidence or narrow the PR's declared intention.

### 16.5 Quality dimensions of a story

Do not turn these into a score. Use them as review questions:

| Dimension | Review question |
| --- | --- |
| Fidelity | Does the story describe the user/problem contract rather than merely the implementation? |
| Completeness | Does every mandatory intention claim have evidence? |
| Continuity | Do adjacent steps use the same producer-owned representation? |
| Discrimination | Would plausible wrong implementations fail? |
| Preservation | Are unchanged/default/compatibility promises protected? |
| Layer adequacy | Is CPU, NPU, multi-rank, checkpoint, numerical, or performance evidence used where needed? |
| Durability | Which proof is permanent CI coverage versus one-time PR evidence? |
| Readability | Can a reviewer understand each testcase's contribution before reading its fixture details? |

### 16.6 Working conclusion

Adopt the story as the central structure, with this wording:

> After the basic PR description, every non-trivial PR must present a test story that traces each
> atomic intention claim through its runtime handoffs to observable behavior. Each testcase must
> explain its role, activation, oracle, rejected fault, handoff, evidence boundary, and result.
> The story covers the PR intention only when every mandatory claim passes the proof gate and no
> plausible counterexample can pass the evidence while violating the intention.

This phrasing preserves the human advantage of narrative while retaining the rigor needed for
automatic review and testcase generation.

## 17. Proposition review model and two entry modes

### 17.1 First correction: code does not reveal intention literally

Reserve **intention** for a human or change-level purpose. From testcase code, extract a
**literal proposition**: the narrowest statement directly supported by its setup, execution, and
assertions.

Keep three statements separate:

```text
L(T)  literal proposition observed by testcase T
G(T)  generalized contract apparently suggested by T, if any
R     proof obligation required by the PR story
```

`L(T)` is extracted from code. `G(T)` is an inference from names, parameterization, surrounding
tests, documentation, or the API contract. `R` is derived from the effective PR intention. A
review must not silently replace one with another.

Quantifiers matter. Three parameter rows do not literally prove an `if and only if` statement
over every possible configuration. The literal statement must say "for the three tested cases"
unless the domain is genuinely exhaustive or generated systematically.

### 17.2 The two entry modes

The same model supports drafting and reviewing by entering from opposite sides.

#### Draft mode -- top down

```text
PR intention
  -> required proof obligation R
  -> scenario + oracle + fault + evidence layer
  -> draft testcase T
  -> re-extract literal proposition L(T)
  -> compare L(T) with R
```

The re-extraction is mandatory. It catches a common drafting failure: the author planned a strong
test but implemented assertions that establish only a helper-level detail.

#### Review mode -- bottom up

```text
existing testcase T
  -> extract literal proposition L(T)
  -> optionally infer generalized contract G(T)
  -> identify required proof obligation R
  -> compare L(T), G(T), and R
  -> retain, narrow, strengthen, supplement, split, or remove
```

Both modes converge on the same proposition-adequacy gate. This permits one core model rather
than separate drafting and review theories.

### 17.3 Literal proposition extraction

Write `L(T)` before discussing why the testcase exists. Include only:

1. the concrete or parameterized preconditions actually constructed;
2. the production operation actually executed;
3. the observations actually asserted;
4. the exact tested domain and evidence layer.

Do not include:

- downstream behavior that was not executed;
- an author's presumed motive;
- universal language unsupported by the cases;
- an oracle-quality judgment;
- a claim that a mock represents the real runtime unless that handoff is checked;
- a PR-level conclusion.

This first description should be deliberately plain. Oracle, fault sensitivity, limitations, and
story role are added during review; they should not obscure what the testcase literally does.

### 17.4 Proposition-adequacy gate

Review the extracted proposition using these questions:

| Gate | Question |
| --- | --- |
| Extractability | Can two reviewers derive materially the same `L(T)` from the code? |
| Fidelity | Does the description use no stronger verbs, scope, or quantifiers than the assertions? |
| Focus | Is there one primary behavioral proposition, even if several causal assertions support it? |
| Validity | Do activation, observation, and oracle actually establish `L(T)`? |
| Relevance | Does `L(T)` map to a required PR claim, preservation contract, or named risk? |
| Discrimination | Which plausible wrong implementation would fail? |
| Layer adequacy | Can this CPU/mock/single-rank layer observe the required behavior? |
| Story connection | What prior fact does the test consume and what later claim may rely on it? |

A testcase can pass the first six gates and still be insufficient for the PR because it is at the
wrong semantic height. A precise helper test is useful seam evidence, but it cannot prove that a
runtime feature is activated.

### 17.5 Review outcomes

Use one of these outcomes rather than simply calling a testcase good or bad:

| Outcome | Meaning |
| --- | --- |
| Retain as core evidence | `L(T)` directly establishes a required proposition at the right layer |
| Retain as supporting evidence | `L(T)` protects a useful seam but needs a higher-layer bridge |
| Narrow or rename | The implementation is useful but its name/description overclaims |
| Strengthen | The intended proposition is appropriate but assertions or cases are too weak |
| Supplement | This test should remain, but another layer or handoff test is required |
| Split | Unrelated primary propositions make failures or purpose ambiguous |
| Remove | The proposition is redundant, irrelevant, circular, or protects an implementation accident |

Improving a description may only make `L(T)` more accurate. Establishing a stronger proposition
requires changing or adding executable evidence.

### 17.6 Virtual review -- Muon compile helper test

Existing testcase:

```python
@pytest.mark.parametrize(
    ("enable", "components", "expected_options"),
    [
        (False, ["muon"], (False, None)),
        (True, ["loss"], (False, None)),
        (True, ["loss", "muon"], (True, "test-backend")),
    ],
)
def test_muon_compile_is_selected_from_compile_config(...):
    ...
```

#### Literal proposition `L(T1)`

> For three configurations using backend `"test-backend"`,
> `get_muon_compile_options()` returns `(False, None)` when global compile is disabled despite a
> Muon component, returns `(False, None)` when compile is enabled without a Muon component, and
> returns `(True, "test-backend")` when compile is enabled with both loss and Muon components.

That is all the testcase literally establishes.

#### Inferred generalized contract `G(T1)`

> The helper enables Muon compile only when global compile is enabled and `"muon"` is a selected
> component; when enabled, it returns the configured backend.

This inference is reasonable because the three rows independently vary the two gates and the
documentation describes the same contract. It remains distinct from the observed three cases.

#### Required PR proposition `R1`

> A Muon compile configuration reaches the constructed optimizer, compiles only
> `zeropower_via_newtonschulz5()` using the selected backend with `fullgraph=True` and
> `dynamic=True`, and the optimizer's LMO uses the returned compiled callable.

`L(T1)` does not establish `R1`.

#### Verdict

```text
Clarity of literal proposition: good
Helper fault discrimination:   useful
PR-feature activation:          absent
Story role:                     supporting seam evidence
Name accuracy:                  overclaims runtime selection
```

The earlier narrative description was too busy because it mixed `L(T1)`, oracle commentary,
mutation analysis, and story limitations before plainly saying what the helper returned. Calling
the test the story's selector also gave it too much semantic authority.

#### Virtual improvement of T1

Rename it to describe the helper contract:

```python
test_get_muon_compile_options_requires_enable_and_muon_component
```

Retain the two independent negative rows and positive row. Add only branches owned by the
declared contract:

- missing compile configuration returns disabled;
- an enabled Muon configuration with no explicit backend selects the documented `inductor`
  default.

Do not add speculative permutations merely because the implementation happens to tolerate them.
After this revision, T1 is still a helper test. The extra rows improve the helper contract; they
do not elevate it into runtime activation evidence.

### 17.7 Virtual addition -- the missing activation testcase

Add one component-level testcase with the primary proposition:

> Building Muon with an enabled Muon compile configuration causes LMO to execute the callable
> returned by `torch.compile` for `zeropower_via_newtonschulz5()` with the declared backend,
> `fullgraph=True`, and `dynamic=True`.

Possible name:

```python
test_muon_build_uses_compiled_newton_schulz_when_enabled
```

Design:

1. Monkeypatch `torch.compile` with a recorder that returns a sentinel wrapper.
2. Build through the real `MuonHybridOptimizersContainer.Config.build()` path using an enabled
   Muon compile configuration.
3. Retrieve the constructed `DistributedMuon` and call LMO on a fixed gradient.
4. Assert the recorder received the Newton-Schulz function, configured backend,
   `fullgraph=True`, and `dynamic=True`.
5. Assert the returned sentinel wrapper was invoked by LMO.
6. Compare the result with the eager Newton-Schulz-plus-normalization reference.

These assertions support one causal proposition: the configured compiled callable becomes the
callable used for the optimizer calculation. They are not unrelated reasons to split the test.

This testcase rejects all of the following implementations:

- the configuration helper returns the correct tuple but the builder drops it;
- the constructor compiles the wrong function;
- the constructor omits the backend or compile options;
- `torch.compile` returns a callable but LMO continues calling the eager global function;
- the compiled handoff changes the component result.

It remains a CPU construction/handoff test when the sentinel compiler is used. Actual NPU
compilation, dynamic-shape reuse, numerical stability, and performance require separate evidence.

### 17.8 Virtual review -- legacy LMO equivalence test

Existing testcase:

```python
def test_muon_lmo_matches_legacy_2d():
    ...
    expected = normalise_grad(zeropower_via_newtonschulz5(grad, ...), ...)
    optimizer._zeropower_fn = zeropower_via_newtonschulz5
    actual = optimizer.lmo(grad, ...)
    assert_close(actual, expected)
```

#### Literal proposition `L(T2)`

> For one fixed-seed `4 x 8` gradient and one set of LMO arguments, an LMO instance whose
> `_zeropower_fn` is the eager Newton-Schulz function returns the same tensor as explicitly
> applying that function and then `normalise_grad()`.

#### Verdict

This is a clear preservation test for refactoring LMO from a static method to an instance method.
Its oracle is the prior function composition, not a mathematical proof of Newton-Schulz and not
compile evidence. Retain it as supporting preservation evidence and rename it only if necessary
to emphasize the refactor scope, for example:

```python
test_lmo_instance_refactor_preserves_legacy_2d_composition
```

Because the refactor affects both supported dimensional branches, either parameterize the legacy
comparison for representative 2D and 3D inputs or add an equivalent 3D preservation case. The
activation testcase in section 17.7 should independently prove that LMO uses the replaceable
callable; T2 should remain focused on preserving eager composition.

### 17.9 Revised minimum evidence inventory before story decomposition

After proposition review, the required evidence inventory is:

| Candidate proof area | Primary proposition | Evidence role |
| --- | --- | --- |
| Configuration | The helper derives enabled/backend options for declared config cases | supporting seam |
| Runtime activation | Real Muon construction uses the compiled Newton-Schulz callable when enabled | core activation |
| Disabled behavior | Disabled construction does not call `torch.compile` and retains eager LMO | default preservation |
| Eager calculation | Instance LMO preserves legacy 2D and 3D eager composition | refactor preservation |
| Parent behavior | Compile-disabled expert updates match the parent under dtype contrast | incidental-change protection |
| NPU equivalence | NPU compiled/eager outputs and updates agree across supported dynamic shapes | backend/numerical proof |
| Training outcome | Deterministic training loss and `grad_norm` match the named baseline | merge numerical evidence |
| Performance outcome | Named metrics meet declared thresholds against a named baseline | merge performance evidence |

This is deliberately called an evidence inventory, not a story. Runtime activation is the first
proposition that reaches the feature named by the PR; the configuration helper only diagnoses an
earlier seam. Section 18 decomposes this inventory into connected proof stories.

### 17.10 Core design decision to carry into rules and skills later

The future rules and skill should share one proposition model and expose two entry commands or
workflow modes:

```text
draft testcase:  derive R -> design T -> extract L(T) -> compare -> narrate
review testcase: extract L(T) -> locate R -> compare -> improve -> narrate
```

Both must produce the same artifacts:

1. literal proposition;
2. optional generalized contract, explicitly labeled as inference;
3. required PR proof obligation;
4. proposition-gate result;
5. review outcome and proposed executable change;
6. accepted role in the larger test story.

This model should be settled before encoding instructions in `.agents/rules/testing.md` or the
test-review skill.

## 18. Story decomposition and readable claim references

### 18.1 The missing structural step

The Muon evidence inventory exposed a gap in the model. Moving directly from reviewed
propositions to one ordered story assumes that a PR has one causal narrative. A common feature PR
instead contains several independently falsifiable proof questions.

Add story decomposition before narration:

```text
literal testcase propositions
  -> proposition review and improvement
  -> identify independent terminal claims
  -> group propositions into proof stories
  -> establish bridges between stories
  -> compare the complete story bundle with the PR intention
```

Propositions belong in one story when they advance the same terminal claim through a real causal
dependency. Sharing a feature, module, or commit is not sufficient.

### 18.2 Muon as a bundle of stories

#### Configuration-to-runtime activation

```text
compile configuration
  -> derive enabled state and backend
  -> real Muon construction installs the compiled Newton-Schulz callable
  -> LMO executes that callable

disabled configuration
  -> no torch.compile call
  -> LMO remains eager
```

Terminal claim:

> User configuration controls which Newton-Schulz implementation the real optimizer executes.

#### Semantic and numerical equivalence

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

The stories need an explicit bridge: the compiled implementation measured in the equivalence
story must be constructed through the production configuration path established by the
activation story. Otherwise, direct invocation can prove compiled/eager equivalence while the
real configuration never activates that implementation.

#### Performance

Performance is independently falsifiable and has a different oracle. Do not combine
"deterministic numerics pass" and "performance meets the target" in one proposition. Treat
performance as a third story when the PR makes that claim:

```text
production activation path
  -> compiled workload after warmup
  -> declared throughput/latency/memory metric
  -> comparison with named baseline and acceptance threshold
```

The PR evidence case is therefore a connected bundle of proof stories, not necessarily one large
story.

### 18.3 Positional labels harm readability

Labels such as `M3`, `M3a`, and `M7b` encode outline position, not meaning. They become unstable
when a proposition is split, moved to another story, or when a missing step is inserted. They
also force the reader to repeatedly look up what each label means.

Apply these rules:

1. Write the narrative using semantic step titles and testcase names, not identifiers.
2. Assign identifiers to atomic claims only when traceability machinery needs them.
3. Namespace identifiers by story and use semantic names rather than sequence numbers.
4. Do not encode parent/child structure using `a`, `b`, or decimal suffixes.
5. Keep identifiers stable when presentation order changes.
6. Do not give a testcase a second artificial ID when its function name already identifies it.

For example, prefer this review matrix:

| Story | Claim key | Proposition |
| --- | --- | --- |
| Activation | `config-selection` | Declared compile settings derive the Muon enable state and backend |
| Activation | `runtime-use` | Real Muon construction installs and uses the compiled callable |
| Activation | `disabled-default` | Disabled construction makes no compile call and remains eager |
| Equivalence | `eager-preservation` | Instance LMO preserves the legacy eager composition |
| Equivalence | `compiled-equivalence` | Activated NPU compiled results match eager over supported shapes |
| Equivalence | `training-equivalence` | Deterministic training loss and grad_norm match the baseline |
| Performance | `performance-acceptance` | Named metrics meet declared thresholds against a named baseline |

Within prose, write:

> The activation story begins with configuration selection. Real construction must consume that
> result and use the compiled callable. The disabled branch must remain eager.

Do not write:

> M1 feeds M2, while M3 proves the alternative branch.

If a machine-readable globally unique key is eventually needed, combine the story namespace and
semantic claim key outside the prose, for example `activation/runtime-use`. The key exists for
traceability; it should not become the language in which humans must understand the story.

## 19. Two-ended story reconciliation

### 19.1 Core model

Test design and test review are two entry points into the same reconciliation problem:

```text
Top down: PR idea -> required proof stories -> verifiable chapters ----\
                                                                   reconcile
Bottom up: testcase code -> literal propositions -> evidence stories --/
```

The two sides should be developed independently enough that the PR description does not cause a
reviewer to overstate what the assertions prove, and existing tests do not cause the required
behavior to be defined around whatever happens to be covered.

### 19.2 Top-down requirement tunnel

Begin with the effective PR intention, including the issue/reproducer, promised behavior, diff,
affected callsites, preservation contracts, supported domain, and numerical or performance
claims.

Then:

```text
PR idea
  -> independently falsifiable terminal claims
  -> one or more proof stories
  -> small verifiable chapters
  -> required propositions and evidence conditions
```

Each chapter should name:

- one primary behavioral proposition;
- controlled stimulus or precondition;
- independently observable result;
- smallest discriminating scenario set;
- plausible fault it must reject;
- required CPU/NPU/multi-rank/checkpoint/numerical/performance layer;
- input and output handoffs to adjacent chapters.

"Small" does not mean forcing every claim into one testcase or a few scalar inputs. It means the
chapter is independently falsifiable and its inputs and observations can be controlled. A
distributed collective or convergence claim must not be decomposed until a CPU mock appears to
prove something that only the real execution layer can establish.

### 19.3 Bottom-up evidence tunnel

For each drafted or existing testcase:

```text
testcase implementation
  -> literal proposition from setup, execution, and assertions
  -> proposition-adequacy review
  -> accepted, revised, or proposed executable evidence
  -> grouping into causally connected evidence stories
```

Extract the literal proposition before assigning a grand story role. Preserve its exact cases,
quantifiers, observation, and evidence layer. Then review its oracle, activation, fault
sensitivity, semantic height, handoffs, and limitations.

Group accepted propositions by the terminal claim they actually advance. Do not force tests into
one narrative merely because they live in the same file or PR.

### 19.4 Meeting in the middle

Reconcile required chapters with literal evidence propositions. This is graph matching, not
test-count matching and not necessarily one-to-one:

- one chapter may require several testcases or artifacts;
- one causally focused testcase may establish several inseparable observations for one chapter;
- a testcase may support more than one story, but its literal proposition must not be counted as
  stronger evidence on the second use;
- adjacent chapters require compatible producer/consumer handoffs.

A match is sufficient only when the evidence establishes the required proposition over the
necessary scenario domain, rejects the relevant fault, uses an adequate oracle and execution
layer, and connects to adjacent chapters through the real representation or a checked contract.

### 19.5 Reconciliation outcomes

| Middle result | Interpretation | Action |
| --- | --- | --- |
| Required chapter and evidence proposition match | The chapter is supported at the stated scope | Retain and connect it in the story |
| Required chapter has no evidence | A promised behavior or risk is unprotected | Draft a testcase or request an artifact |
| Evidence is weaker or lower-layer than required | The test is relevant but insufficient | Strengthen it or add a bridge/higher-layer test |
| Evidence proposition has no required chapter | It may be decorative, redundant, or reveal an omitted requirement | Investigate before retaining or removing |
| Required and observed propositions conflict | Test, implementation, or PR specification disagrees | Report and resolve the contradiction |
| Both propositions match locally but handoffs differ | The suite proves disconnected lookalike paths | Add a bridge or use real producer output |
| Numerical/performance evidence lacks activation proof | The result may exercise the wrong configuration | Connect the run to the production activation story |

An unmatched bottom-up proposition is not automatically useless. It may reveal an incidental
behavior change in the diff that the PR description omitted. In that case, revise the top-down
effective intention and add a preservation or boundary chapter.

### 19.6 Definition of completion

The two tunnels meet when:

1. every mandatory top-down chapter has adequate bottom-up evidence;
2. every retained testcase proposition maps to a named claim, risk, or preservation contract;
3. every changed runtime handoff is continuous across the evidence stories;
4. independent stories are connected wherever one assumes activation or output established by
   another;
5. unresolved partial matches, unsupported scope, and merge-only evidence remain explicit;
6. a hostile implementation cannot pass the evidence bundle while violating the effective PR
   intention.

The final review artifact is therefore not just a generated test list. It contains the required
story graph, the observed evidence graph, and the reconciliation between them.
