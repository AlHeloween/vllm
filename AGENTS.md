## Reasoning Guideline

1. State before reasoning.
2. Decompose before expanding.
3. Verify before reducing.
4. Use k-medoids, not semantic averaging.
5. Reference outranks inference.
6. For safety-critical tasks, arbitrate many standards but answer under one governing standard only.
7. Never blend incompatible normative regimes.
8. Preserve semantic traceability.
9. Let oracle output decide correctness.
10. Emit a clean next state.

# Quick-Start

**Contents:** [Before any activity](#before-any-activity) - [Response and traceability](#response-and-traceability) - [Build and run](#build-and-run) - [Code quality](#code-quality--standards) - [References](#references)

---

## Absolute rules (CRITICAL)

1) **Reports narrowing is strictly prohibited**:
   - Do not reduce a report to a narrower success claim than the user requested.
   - Do not present a single-stage, single-layer, or partial-surface result as if it proves the whole system.
   - If a report covers only one narrowed scope, state that scope explicitly and list what remains unchecked in the same report.
   - Do not stop at a narrowed report when the requested task is broader; continue until the full requested surface is checked or blocked.

2) **Completion validation is mandatory**:
   - Before saying a job is complete, execute and check the exact requested surface.
   - If the task affects an executable, launch that executable through the intended entry path before reporting completion.
   - If the task affects a GUI executable, perform a real no-args GUI launch check, not only compilation.
   - If the task affects tests, run the relevant test target before reporting completion.
   - If the task affects a CLI path, run the relevant command path before reporting completion.
   - If a launch/check cannot be performed, state that exact blocker and do not report the broader job as complete.

3) **Working-directory scope**: while the current working directory is `external/vllm/`, do not modify any files outside `external/vllm/**` unless the user explicitly asks to change scope.

4) **Chat/log hygiene**: do not ask the user to paste logs or long file contents when you have filesystem access.
   - Prefer reading local artifacts directly (for example `logs/**`, `test_data/**`).
   - Do not dump debug logs into chat (especially repeated per-layer lines). Write them to a dedicated file and reference it.
   - If a snippet is required for discussion: include at most ~20 lines, numbered, and only after you've summarized what the snippet proves.

5) **Process execution stability**: prefer `cmd_runner` for most unknown, long-running, noisy, or crash-prone program calls.
   - Use `cmd_runner.py` (or the repo `cmd_runner` workflow) by default when debugging, compiling, or running LLM CLI/server binaries.
   - Only run programs directly when they are short, well-understood, and unlikely to flood output or hang.
   - If permissions, policy, or environment restrictions block the intended execution path, report that blocker to the user immediately with the exact failing command path and the chosen fallback.

6) **Use PowerShell, not cmd, for Windows command execution**:
   - The `cmd` shell is strictly prohibited for any command execution.
   - Use PowerShell (`powershell -Command ...` or `powershell -File ...`) for all Windows shell commands. Prefer `powershell -c "..."` for simple commands.
   - PowerShell can call `.bat` files and `.cmd` scripts directly (e.g., `powershell -Command "& { .\build_all.bat }"`).
   - For environment initialization scripts (like `tools\init_msvc.cmd`), use `cmd /c` only when absolutely required by the script's design, and wrap that call in PowerShell (e.g., `powershell -Command "cmd /c 'tools\init_msvc.cmd'"`).
   - This rule eliminates cmd quoting issues, improves error handling, and ensures consistent cross-session behavior.

7) **Avoid Windows system filenames**:
   - Windows does not have `/dev/null`. Never create files named `nul`, `null`, `con`, `prn`, `aux`, `com1`-`com9`, `lpt1`-`lpt9`, or any other Windows reserved device names.
   - When redirecting output to discard it on Windows, use `powershell -Command "..." > $null` or `... 2>&1 > $null`.
   - When testing compilation output, use a temporary file with a proper extension (e.g., `test_output.obj`, `test_output.dll`) or discard output using PowerShell redirection.
   - Creating files with system names causes errors and looks unprofessional.

8) **Python version consistency**: use Python `3.13` only.
   - All direct `python` execution and all `uv`-based execution must resolve to Python `3.13`.
   - If you detect version drift, interpreter mismatch, or scripts/tools bound to another Python version, correct that misalignment before continuing.

9) **Debug isolation rule**: if `_debugging.md` exists at the repo root, it is the canonical registry of files and paths under active debugging.
   - Read `_debugging.md` before touching any implementation area that may have both a main path and a debug path.
   - If a file or subsystem is listed in `_debugging.md` as unresolved, do not modify the corresponding main-project implementation path.
   - During unresolved debugging, all experimental edits must stay in the registered debug path only.
   - Move fixes from debug to the main path only after the debug entry is explicitly marked resolved in `_debugging.md`.

10) **Explicit performance directives**:
    - When writing math, array, or tensor libraries in C++/CUDA, prioritize execution speed over DRY principles in performance-critical paths.
    - Do not nest function calls for simple index calculations unless using explicit inline directives or an already-proven low-overhead pattern.
    - Keep hot-path indexing, stride calculation, and data access visibly direct when that avoids extra call overhead.

## Response and traceability

Rules to follow in every response:

1. Act as the most qualified expert.
2. Answers: numbered schemas, variables or equations, accurate, factual, well-structured, non-repetitive, multi-perspective, matched to content time frame. Mark as: Exact (verifiable), Inferred (deduced), Hypothetical (scenario), Guess (low confidence), or Unknown.
3. Every answer must contain an explicit task view:
   - `Current tasks` = the active requested surface or the exact remaining scope being worked.
   - `Completed tasks` = the exact completed work, not a vague success claim.
   - `Inferred next tasks` = the logically next steps after the completed work.
   - Do not say `implemented`, `done`, or similar without listing what exactly was completed.
   - If only part of the requested surface is complete, state the remaining tasks in the same answer.
4. Report physical harms in units or variables; no safety procedures unless asked.
5. Traceability: maintain clear semantic traceability between tasks and outcomes.

## Build and run

- **vLLM build**: Follow the official vLLM build instructions. For Windows, use the community fork `SystemPanic/vllm-windows`.
- **CMake**: Primary build system for C++/CUDA extensions.
- **Python package**: Install with `pip install -e .` or `uv pip install -e .`.
- **Testing**: Run `pytest tests/` for Python tests.

## Code quality and standards

### General style
- Style: follow existing project conventions; keep changes consistent with nearby code.
- Docstrings: write clear docstrings for new public Python functions where applicable.
- Type hints: add Python type hints when editing Python utilities.

### CUDA/C++ standards
- Follow vLLM's existing code style for CUDA kernels.
- Use consistent naming: snake_case for functions/variables, PascalCase for classes/structs.
- Include proper error checking for CUDA API calls.

### Logic and transparency
- Modular: keep interfaces small and explicit.
- Provenance: prefer repo docs and codebase inspection; avoid guessing API usage.
- Clarity on uncertainty: mark assumptions or unverified code clearly.

### Security and configuration
- No hardcoded secrets.
- Logging: do not log sensitive data.

## Command-line search rules

- File content search: use `rg` (ripgrep). Do not use `grep`/`find` for repo search.
- File listing/name search: use `fd` only (do not use `rg --files`).
- Output filtering: `grep` is allowed only for piping non-repo command output.

## References

- `README.md` - project overview
- `pyproject.toml` - Python package configuration
- `setup.py` - legacy setup configuration
- `CMakeLists.txt` - C++/CUDA build configuration
- `docs/` - documentation