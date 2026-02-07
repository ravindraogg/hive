# Hive Repository Audit Notes (Initial Pass)

This audit summarizes initial architectural and security observations after reviewing runtime
execution flow, tooling, and agent routing logic. Findings aim to improve reliability, security
posture, and contributor onboarding.

## Scope

- Repository: https://github.com/adenhq/hive
- Focus areas: architecture, setup, security, static analysis, and developer experience.

## Summary of Findings

### High Priority Bugs

- Legacy single-entry execution uses the deprecated `FileStorage`.
  - Location: `core/framework/runtime/core.py`, `core/framework/storage/backend.py`,
    `core/framework/builder/query.py`.
  - Evidence: `FileStorage.save_run()` is a no-op, and `Runtime.end_run()` invokes it.
    `BuilderQuery` loads runs from `FileStorage`, so no persistence occurs.
  - Impact: breaks debugging history, disables reproducibility, and violates documentation
    claims about file-based storage.
- Documentation claims file-based run storage, but the underlying backend is marked deprecated
  and no longer writes (mismatch between docs and runtime behavior).

### Security Vulnerabilities

- Shell command execution tool uses `subprocess.run(..., shell=True)` without
  allowlisting or command validation, increasing RCE risk even inside the workspace
  sandbox.
  - Risk layers: injection via tool arguments, malicious agent-generated commands,
    and lateral movement inside the workspace once a command can run.
  - Suggested mitigation: adopt a structured command schema (argv arrays) with
    explicit allow/deny lists, and default to `shell=False`.
- LLM-powered routing uses raw node output and shared memory inside prompts, which
  can embed untrusted tool output and enable prompt injection into routing decisions.
- Sandbox timeouts are disabled on platforms without `SIGALRM`, allowing unbounded
  execution when running dynamic code.

### Performance Risks

- LLM-based edge routing adds model calls to every `llm_decide` edge, which can
  inflate latency and cost without batching.
- Shared memory and runtime logs can grow without explicit size limits or retention
  policies in long-lived executions.

### Missing Features

- Confidence calibration, rule generation, and signal-weighting phases described
  in the architecture are designed but not yet implemented.
- `.env.example` referenced in documentation is not present in the repository,
  causing onboarding friction for tool setup.

### Alignment With Project Goals

- Self-evolving agents require reliable observability and persistence. Current logging
  and persistence gaps reduce the feedback loop needed for evolution, making it harder
  to learn from failures at scale.

### Suggested PR Ideas

- Migrate legacy `Runtime`/`BuilderQuery` to the unified `SessionStore` (or add
  a compatibility shim) so single-entry agents persist runs again.
- Harden `execute_command_tool` with allowlists, deny lists, and `shell=False` by
  default, plus structured args.
- Add prompt-injection defenses (tool output quarantining or structured context)
  for LLM routing and judge prompts.
- Add `.env.example` with documented keys and clarify provider setup in docs.

### Repro Steps

1. Run a legacy single-entry agent using the `Runtime` workflow.
2. Observe expected run persistence (per docs) under the configured storage path.
3. Check `{storage_path}/runs/` for new run artifacts.
4. Result: no saved runs generated because `FileStorage.save_run()` is a no-op.

### Quick Wins for Contributors

- Add tests for command execution restrictions and sandbox timeouts on non-Unix.
- Document single-entry vs multi-entry runtime persistence paths.
- Provide a minimal example agent plus `hive run` walkthrough with real outputs.
