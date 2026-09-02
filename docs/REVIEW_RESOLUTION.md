# Review resolution record

- Repository: `Saber5656/gitquest`
- Pull request: #1
- Parent head observed before this addendum: `8b3e2ba0b814684360b1cd670310b6722e285f96`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTN39Vc6QDZHG`

### Expose stdin for cat-file batch processes

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHG` identifies this contract gap.
- Normative resolution: Extend the hardened process contract with a caller-owned stdin writer/closer, stdout reader, and Wait result for long-lived `git cat-file --batch` sessions; define close ordering and error propagation at the runner boundary.
- Focused verification before resolving this thread: Start one batch process, send multiple object ids through the returned stdin, assert ordered responses and EOF after close, and verify no caller bypasses the hardened runner.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHH`

### Anchor repo activity to scan time, not HEAD time

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHH` identifies this contract gap.
- Normative resolution: Capture one injected `ScanTime` at scan start and evaluate the 90-day activity window as elapsed time from the latest repository activity to that scan time; `HeadTime` is data, never the clock anchor.
- Focused verification before resolving this thread: Run the ghost gate with an old commit and a fixed current clock, then with a recent commit; assert the old repository is disabled and the recent one remains eligible.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHJ`

### Keep repository IDs stable for shallow clones

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHJ` identifies this contract gap.
- Normative resolution: Make the persisted repository identity independent of a shallow boundary: derive it from the canonical repository identity and retain it in state, while a root commit is optional metadata and must not replace the id after unshallowing.
- Focused verification before resolving this thread: Create a depth-1 clone, record state, deepen it, and assert the profile id and XP/tame state are unchanged.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHK`

### Strip valid UTF-8 C1 controls too

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHK` identifies this contract gap.
- Normative resolution: Define sanitizer behavior over decoded runes as well as raw bytes: reject or replace C0/C1 controls, including valid UTF-8 U+0080–U+009F, before values enter rendered output or persisted reports.
- Focused verification before resolving this thread: Feed U+009B and other valid UTF-8 C1 runes in paths and TODO text, then assert sanitized output contains no C0/C1 controls.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHM`

### Add a workflow_call trigger before reusing CI

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHM` identifies this contract gap.
- Normative resolution: Add `workflow_call` to the reusable CI workflow contract and keep its inputs/secrets explicit so the release workflow can invoke the same gates on tags.
- Focused verification before resolving this thread: Parse the workflow files and execute a reusable-workflow validation that confirms the release `uses:` target declares `workflow_call`.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHP`

### Load mutable state only after taking the lock

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHP` identifies this contract gap.
- Normative resolution: For `scan`, `tame`, and `reset`, acquire the profile lock before reading mutable state; read-only commands may use an unlocked snapshot but must not write it.
- Focused verification before resolving this thread: Run concurrent stale-reader/writer scenarios and assert the later locked mutation cannot be overwritten by a pre-lock state snapshot.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHR`

### Scrub Git environment during discovery

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHR` identifies this contract gap.
- Normative resolution: Apply the same Git environment denylist and explicit repository directory to discovery `rev-parse` calls that the hardened runner uses; ambient `GIT_DIR` and `GIT_WORK_TREE` must not select another repository.
- Focused verification before resolving this thread: Set conflicting ambient Git variables while discovering from a fixture and assert the discovered top level is the requested repository.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHS`

### Do not rely on encoding/json for all controls

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHS` identifies this contract gap.
- Normative resolution: Sanitize or custom-escape report JSON string fields for C0/C1 and bidi controls before encoding, and retain the byte-sweep invariant over the final stdout bytes.
- Focused verification before resolving this thread: Emit JSON containing U+009B, bidi controls, and ESC from repository-derived values, then assert the final bytes pass the security sweep and JSON remains parseable.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN39Vc6QDZHT`

### Preserve cancellation reasons for quest events

- Finding: The existing review thread `PRRT_kwDOTN39Vc6QDZHT` identifies this contract gap.
- Normative resolution: Change quest cancellation input to carry a reason per monster id, constrained to `fled` or `tamed`; event construction must preserve that mapping when both reasons occur in one scan.
- Focused verification before resolving this thread: Cancel quests for a mixed set of fled and tamed monsters and assert each `quest_cancelled` event has the matching reason.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.