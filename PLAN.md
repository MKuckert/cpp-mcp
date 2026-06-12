# Project Plan: CLIBridge — Executable Directory Scanner

## 🎯 Objective

Extend `clibridge` so that on startup it recursively scans a user-supplied directory for
executable files and registers each one as an MCP tool. When an agent calls such a tool,
the server spawns the executable, optionally pipes `stdin` data into it, and returns
`stdout`, `stderr`, exit code, and timeout/signal flags as a structured JSON result.
Process-level outcomes (non-zero exit, timeout, signal) are communicated through the
structured response; MCP `isError` is reserved for OS/server-level failures only.

## 🛠 Requirements & Decisions

- **Frameworks:** `cpp-mcp` core library (already in-tree), C++17 `<filesystem>`.
- **Chosen Libraries:** `reproc++` via CMake `FetchContent` for cross-platform process
  spawning with timeout support.
- **Error Handling Strategy:**
  - **Timeout (30 s constant):** Kill process; return normal MCP response with
    `"timedOut": true` and any partial `stdout`/`stderr` collected.
  - **Executable deleted between scan and call:** Throw
    `mcp_exception(invalid_params, ...)` → `isError: true`.
  - **Non-zero exit / signal kill:** Return normal MCP response with the full
    `stdout`/`stderr`, `exitCode`, and `signalKilled: true` fields set. The agent
    interprets these fields; no MCP error is raised.
  - **`stdin` data not consumed by child:** No special handling; pipe closes naturally.
  - **No executables found in directory:** Server starts normally without tools; log a
    warning to `std::cerr`.
  - **Unexpected OS-level reproc failures:** Throw `mcp_exception(internal_error, ...)`
    → `isError: true`.

## 🏗 Implementation Steps

> Status Markers: [ ] Open, [/] In Progress, [x] Completed (By the Reviewer only!)

- [x] **Task 1: Add reproc++ Dependency**
  - **Description:** Add a `FetchContent` block to the clibridge `CMakeLists.txt` that
    downloads and builds `reproc` (tag `v14.2.4` or latest stable). Export the `reproc++`
    target so `clibridge` can link against it. Use `EXCLUDE_FROM_ALL` to avoid polluting
    the default install target. Example snippet:
    ```cmake
    include(FetchContent)
    FetchContent_Declare(
      reproc
      GIT_REPOSITORY https://github.com/DaanDeMeyer/reproc.git
      GIT_TAG        v14.2.4
    )
    set(REPROC++ ON)
    FetchContent_MakeAvailable(reproc)
    ```
  - **Review Criteria:** `script_runner_build_sh` succeeds; `reproc++`
    target is available for linking; no existing targets are broken.

- [ ] **Task 2: Implement `ExecutableScanner`**
  - **Description:** Create `clibridge/executable_scanner.h` (header-only).
    Expose one function:

    ```cpp
    // Returns list of {tool_name, absolute_path} pairs.
    // tool_name = relative path string, e.g. "foo/bar/script.sh"
    std::vector<std::pair<std::string, std::filesystem::path>>
    scan_executables(const std::filesystem::path& scan_root);
    ```

    - Use `std::filesystem::recursive_directory_iterator` (symlink-following disabled
      by default — do not override this).
    - Include only regular files where
      `(status.permissions() & fs::perms::owner_exec) != fs::perms::none`.
    - Derive `tool_name` via `fs::relative(entry, scan_root).generic_string()`.
    - If `scan_root` does not exist or is not a directory, throw `std::runtime_error`.
    - If the result is empty, return empty vector (caller logs the warning).
    - **Note:** On Windows, `owner_exec` is unreliable; add a `#ifdef _WIN32`-guarded
      extension check (`.exe`, `.bat`, `.cmd`, `.ps1`) as a TODO stub.

  - **Review Criteria:** Unit tests cover: normal tree with mixed files, empty dir,
    nested executables, non-executable files excluded, non-existent dir throws.

- [ ] **Task 3: Implement `ProcessRunner`**
  - **Description:** Create `clibridge/process_runner.h` (header-only).
    Expose:

    ```cpp
    struct RunResult {
        std::string stdout_data;
        std::string stderr_data;
        int         exit_code    = 0;     // valid when !timed_out && !signal_killed
        bool        timed_out    = false;
        bool        signal_killed = false;
    };

    RunResult run_process(
        const std::filesystem::path& executable,
        const std::string&           stdin_data,
        std::chrono::milliseconds    timeout = std::chrono::seconds(30));
    ```

    Implementation using `reproc++`:
    1. Build `reproc::options` with `redirect.in/out/err = REPROC_REDIRECT_PIPE`.
    2. Configure stop actions: `REPROC_STOP_TIMEOUT` (timeout ms) then
       `REPROC_STOP_KILL` (5 s grace).
    3. Write `stdin_data` to the process, then close the stdin stream.
    4. Drain stdout and stderr (separate buffers) in a loop until EOF.
    5. Call `process.wait()` / inspect the return status.
    6. Map `REPROC_ETIMEDOUT` → `timed_out = true`.
    7. On POSIX, a negative reproc exit status indicates signal termination →
       `signal_killed = true`. Document this with a comment.
    8. Re-throw any other reproc error as `std::runtime_error` with the
       `reproc::strerror(ec)` message included.

  - **Review Criteria:** Unit tests cover: successful execution + stdout capture,
    non-zero exit code propagated, stdin data fed to child and echoed back, timeout
    triggers kill with partial output, missing executable path returns OS error.

- [ ] **Task 4: Wire Scanner + Runner into `clibridge/main.cpp`**
  - **Description:** Refactor `main.cpp`:
    1. Parse `--dir <path>` from `argv` (simple manual parse; no extra library).
       If `--dir` is absent, print usage to `stderr` and `exit(1)`.
    2. On POSIX, ignore `SIGPIPE` before doing anything else:
       ```cpp
       #ifdef __unix__
       signal(SIGPIPE, SIG_IGN);
       #endif
       ```
    3. Call `scan_executables(dir)`. If empty, write a warning to `std::cerr` and
       continue (no tools will be registered).
    4. For each `{tool_name, abs_path}` pair, register an MCP tool:
       - Name: `tool_name` (e.g. `"util/convert.sh"`).
       - Description: `"Executes " + tool_name`.
       - One optional string parameter `"stdin"` — data to pipe into the process.
       - **Lambda captures: all variables (`abs_path`, `tool_name`) must be captured
         by value** to avoid use-after-free after the registration loop ends.
    5. Each tool handler:
       a. Extracts optional `"stdin"` param (default `""`).
       b. Checks `std::filesystem::exists(abs_path)` — if false, throws
       `mcp_exception(invalid_params, "Executable no longer exists: " + abs_path.string())`.
       c. Calls `run_process(abs_path, stdin_data)`.
       d. Serialises the result as a JSON object and returns it as the single text
       content item of a normal (non-error) MCP response:
       ```json
       {
         "stdout": "...",
         "stderr": "...",
         "exitCode": 0,
         "timedOut": false,
         "signalKilled": false
       }
       ```
       e. Only throws `mcp_exception(internal_error, ...)` for unexpected OS-level
       reproc failures (e.g. `REPROC_ENOMEM`).
    6. Remove the four hardcoded example tools.
    7. Update server info to `"clibridge"` / `"1.0.0"`.
  - **Review Criteria:** Integration test verifies: (a) stdout captured from a
    trivial script, (b) a script that exits non-zero returns correct `exitCode` and
    `isError` is absent, (c) deleted executable returns `isError: true`.

- [ ] **Task 5: Update `clibridge/CMakeLists.txt`**
  - **Description:** Add `reproc++` to `target_link_libraries` for the `clibridge`
    target.
  - **Review Criteria:** Clean build on macOS and Linux; no linker errors.

- [ ] **Task 6: Tests**
  - **Description:** Add test file(s) under `test/` (matching existing test structure).
    - Unit tests for `scan_executables` (Task 2 criteria).
    - Unit tests for `run_process` (Task 3 criteria).
    - One integration smoke test that:
      1. Creates a **unique temp directory** via
         `std::filesystem::temp_directory_path() / "clibridge_test_<unique_suffix>"`.
      2. Writes a small `echo_stdin.sh` (or `.bat` on Windows) fixture into it and
         marks it executable (`fs::permissions(..., fs::perms::owner_exec)`).
      3. Calls `scan_executables` and `run_process` directly to exercise the tool.
      4. **Cleans up** the temp directory unconditionally in a teardown step
         (RAII guard or explicit `fs::remove_all`).
  - **Review Criteria:** All tests pass under `ctest`; no memory leaks reported by
    AddressSanitizer.

## 🛡 Edge Case & Safety Checklist

- [ ] Recursive scan does not follow symlinks into loops (default iterator behaviour;
      do not set `follow_directory_symlink`).
- [ ] Tool names with `/` path separators are valid MCP identifiers — verify with a
      real MCP client before shipping.
- [ ] Zombie processes: let reproc call `waitpid`; never call it independently.
- [ ] SIGPIPE guarded with `#ifdef __unix__` and placed at the top of `main()`.
- [ ] Lambda captures by value in the tool-registration loop (see Task 4 step 4).
- [ ] Concurrent calls: each invocation spawns an independent process; no shared state.
- [ ] Large output: no artificial cap; `std::string` accumulation. Document that
      callers should be mindful of very large outputs.
- [ ] Windows: `owner_exec` unreliable — add extension-based fallback as TODO stub.

## 📝 Review Log (Mode 1: Plan Review)

- **Round 1:** REJECTED — lambda capture lifetime bug; ambiguous error semantics (dual
  isError approach); missing SIGPIPE platform guard; missing temp-dir spec in tests;
  header-only compile-time note missing.
- **Round 2:** [Pending]
- **Round 3:** [N/A]

## 🚦 Final Status (Mode 2: Code Review)

- **Task 1:** APPROVED. `reproc` (v14.2.4) integrated cleanly via CMake FetchContent. Option cache variables are set appropriately, and targets are available without breaking existing dependencies.
