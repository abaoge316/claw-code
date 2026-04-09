# Turn Failure Fallback to GLM-4.7 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a model request fails in the REPL, print the error, switch the active model to `glm-4.7`, and keep the session alive without resending the failed turn.

**Architecture:** Keep the fallback policy at the CLI layer. The request path should remain unchanged; only the error handler should decide whether to switch models. The fallback must be best-effort and must never reissue the failed prompt.

**Tech Stack:** Rust, existing CLI/runtime crates, existing CLI integration tests.

---

### Task 1: Add a recoverable turn-failure handler

**Files:**
- Modify: `crates/rusty-claude-cli/src/main.rs`
- Test: `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs`

- [ ] **Step 1: Write the failing test**

```rust
#[test]
fn repl_request_failures_auto_fallback_to_glm_47_without_resending_the_prompt() {
    let temp_dir = unique_temp_dir("repl-fallback-glm-47");
    let config_home = temp_dir.join("home").join(".claw");
    fs::create_dir_all(&config_home).expect("config home should exist");
    fs::write(
        temp_dir.join(".claw.json"),
        r#"{
            "defaultModel":"MiniMax-M2.7",
            "models":[
                {"name":"glm-4.7","baseUrl":"https://open.bigmodel.cn/api/coding/paas/v4"},
                {"name":"glm-4.7-flash","baseUrl":"https://open.bigmodel.cn/api/coding/paas/v4"},
                {"name":"MiniMax-M2.7","baseUrl":"https://api.minimax.io/v1"}
            ]
        }"#,
    ).expect("write project config");

    let mut child = command_in(&temp_dir)
        .env("CLAW_CONFIG_HOME", &config_home)
        .env("OPENAI_API_KEY", "openai-test-key")
        .env("MINIMAX_API_KEY", "invalid-minimax-key")
        .args(["--model", "MiniMax-M2.7"])
        .stdin(std::process::Stdio::piped())
        .stdout(std::process::Stdio::piped())
        .stderr(std::process::Stdio::piped())
        .spawn()
        .expect("claw should launch");

    {
        let stdin = child.stdin.as_mut().expect("stdin should be piped");
        stdin.write_all(b"what is your name?\n/model\n/exit\n").expect("write repl input");
    }

    let output = child.wait_with_output().expect("wait for claw");
    assert_success(&output);

    let stdout = String::from_utf8(output.stdout).expect("stdout should be utf8");
    assert!(stdout.contains("Request failed"), "{stdout}");
    assert!(stdout.contains("Model updated"), "{stdout}");
    assert!(stdout.contains("Current          glm-4.7"), "{stdout}");
    let stderr = String::from_utf8(output.stderr).expect("stderr should be utf8");
    assert!(stderr.contains("401 Unauthorized"), "{stderr}");
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test -p rusty-claude-cli --test cli_flags_and_config_defaults repl_request_failures_auto_fallback_to_glm_47_without_resending_the_prompt -- --exact`
Expected: FAIL until the fallback path exists.

- [ ] **Step 3: Implement the fallback**

Add a `glm-4.7` fallback constant and a `run_turn_recoverable` helper that:
1. prints the original turn error
2. switches to `glm-4.7` best-effort
3. never retries the failed prompt

Use the helper from the REPL submit path and the `/skills` invoke path.

- [ ] **Step 4: Run the test to verify it passes**

Run: `cargo test -p rusty-claude-cli --test cli_flags_and_config_defaults repl_request_failures_auto_fallback_to_glm_47_without_resending_the_prompt -- --exact`
Expected: PASS and stdout shows the fallback model switch.

- [ ] **Step 5: Validate the existing model registry behavior**

Run:
`cargo test -p rusty-claude-cli --test cli_flags_and_config_defaults model_command_lists_builtin_registry_when_project_config_is_absent -- --exact`
`cargo test -p runtime --lib falls_back_to_builtin_model_registry_when_no_project_models_are_configured -- --exact`

Expected: PASS, proving the fallback switch did not break registry loading.
