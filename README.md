# pvm-verify-profiles

This repository is for external teams to submit a profile and `.so` for equivalence verification.

Use this repo to submit your team profile and `.so` for quick verification.

Only `compute` suite is supported for external teams.

## Quick Start
This repo includes a prebuilt `so_probe_cli` binary (Linux x86_64) at the repo root.

1. Create your team folder under `profiles/`.

```bash
mkdir -p profiles/<team_name>
```

2. Setup from `polkajam`:

```bash
cp profiles/polkajam/config.json profiles/<team_name>/config.json
cp /path/to/your/libxxx.so profiles/<team_name>/<your_lib_name>.so
```

3. Edit `profiles/<team_name>/config.json`:

- Set `group_name` to your team name.
- Keep `suites` as `["compute"]`.
- Set `target_version` to your runtime version (`0.7.2` or `0.8.0`).
- Set `register_mapping`:
   1. If your register mapping matches an existing one, use `default` or `polkavm`.
   2. If it does not match, add your own mapping file at `regmaps/<name>.yaml`, then set `register_mapping` to `<name>`.
- Set `shared_library` to your `.so` filename (example: `<your_lib_name>.so`).

4. Run verify:

```bash
./so_probe_cli --profile profiles/<team_name>/config.json
```

Pass condition:

```text
[result] PASS
```

Shared library requirement:

- Your `.so` only needs to export the `GetX86Bytes` symbol.
- Platform constraint: in this repo, `.so` means a Linux ELF shared library.
   It must match the `so_probe_cli` platform/architecture (currently Linux x86_64).
- If you develop on macOS/Windows, you still need to produce a Linux x86_64 `.so`
   and run `so_probe_cli` inside a Linux environment (VM/container/CI).

## Background: How We Use Your Profile + .so for SMT Verification
This section is background context and is not required to run the Quick Start.

We run formal equivalence verification in `pvm-x86-inst-equivalence` using your profile and `.so`.

End-to-end flow:

```mermaid
flowchart TD
    A[Load profile config] --> B[Load team .so from shared_library]
    B --> C[Pick compute-suite instruction cases]
    C --> D[Interpreter semantics for each case]
   C --> E[Call .so (GetX86Bytes)]
    E --> F[Recompiler semantics from x86]
    D --> G[SMT equivalence query]
    F --> G
    G -->|UNSAT| H[Equivalent: case PASS]
    G -->|SAT| I[Mismatch: case FAIL + counterexample]
    H --> J[Aggregate verification report]
    I --> J
```

Per-case logic:

1. Input:
   - `opcode + operands` test case
   - profile settings (`target_version`, `register_mapping`, `skip_instructions`, `shared_library`)
2. Recompiler path:
   - call your `.so` (`GetX86Bytes`) -> x86 bytes
   - decode to symbolic semantics
3. Interpreter path:
   - build expected PVM semantics for the same case
4. Solver check:
   - ask SMT solver whether both semantics are equivalent
5. Output:
   - `UNSAT` => equivalent (pass)
   - `SAT` => mismatch (fail, with concrete counterexample state)

Report result:

- If all checked cases are equivalent: overall PASS
- If any case is not equivalent: overall FAIL and include mismatch details
