# Windows CLI Remote-Control Compatibility Plan

## Goal

Restore a Windows-native CLI path for:

```powershell
codex remote-control
```

while keeping the rest of the current Codex codebase, release behavior, and improvements from the latest fork baseline.

This should reproduce the practical behavior that worked in Codex CLI `0.130.0`: starting a foreground app-server with remote-control enabled, without relying on daemon lifecycle support or Unix-only daemon readiness probes.

## Non-goals

- Do not reimplement Windows daemon lifecycle support.
- Do not change `codex remote-control start` or `codex remote-control stop`.
- Do not change macOS/Linux behavior.
- Do not replace the officially supported Codex App remote-connection flow.
- Do not bypass authentication, pairing, enrollment, or remote-control authorization flows.

## Current problem

The modern CLI path for bare `codex remote-control` starts a foreground app-server on a local control socket and then calls into the app-server daemon helper to enable/readiness-check remote control.

That helper still depends on daemon platform support and currently rejects non-Unix platforms with an error equivalent to:

```text
codex app-server daemon lifecycle is only supported on Unix platforms
```

This means the Windows-native CLI path is structurally blocked even though the app-server runtime itself already has the pieces needed to run remote-control in foreground mode.

## Desired behavior

On Windows only:

```powershell
codex remote-control
```

should:

1. Start the app-server in the foreground.
2. Enable remote-control startup mode immediately.
3. Avoid daemon lifecycle APIs.
4. Avoid the daemon Unix-socket readiness probe.
5. Keep running until the user presses `Ctrl-C`.
6. Allow ChatGPT mobile to discover/select the machine through the existing remote-control relay/enrollment path.

On non-Windows platforms, the current implementation should remain unchanged.

## Design

Add a Windows-only implementation of `run_foreground_remote_control(...)` in:

```text
codex-rs/cli/src/remote_control_cmd.rs
```

Use conditional compilation:

```rust
#[cfg(windows)]
async fn run_foreground_remote_control(...) -> anyhow::Result<()> { ... }

#[cfg(not(windows))]
async fn run_foreground_remote_control(...) -> anyhow::Result<()> { ... }
```

The Windows implementation should call:

```rust
codex_app_server::run_main_with_transport_options(...)
```

with:

```rust
AppServerTransport::Off
SessionSource::Cli
AppServerWebsocketAuthSettings::default()
AppServerRuntimeOptions {
    remote_control_startup_mode: codex_app_server::RemoteControlStartupMode::EnabledEphemeral,
    install_shutdown_signal_handler: true,
    ..Default::default()
}
```

This mirrors the old foreground path while using the current app-server runtime field names and types.

## Proposed patch shape

```rust
#[cfg(windows)]
async fn run_foreground_remote_control(
    json: bool,
    arg0_paths: Arg0DispatchPaths,
    root_config_overrides: CliConfigOverrides,
) -> anyhow::Result<()> {
    let runtime_options = AppServerRuntimeOptions {
        remote_control_startup_mode: codex_app_server::RemoteControlStartupMode::EnabledEphemeral,
        install_shutdown_signal_handler: true,
        ..Default::default()
    };

    if !json {
        println!("Remote control is running in Windows foreground compatibility mode.");
        println!("Press Ctrl-C to stop.");
    }

    codex_app_server::run_main_with_transport_options(
        arg0_paths,
        root_config_overrides,
        LoaderOverrides::default(),
        /*strict_config*/ false,
        /*default_analytics_enabled*/ false,
        AppServerTransport::Off,
        SessionSource::Cli,
        AppServerWebsocketAuthSettings::default(),
        runtime_options,
    )
    .await
    .context("foreground app-server exited with an error")
}
```

## Files to inspect before implementation

- `codex-rs/cli/src/remote_control_cmd.rs`
- `codex-rs/app-server/src/lib.rs`
- `codex-rs/app-server-daemon/src/lib.rs`
- `codex-rs/app-server-protocol/src/` remote-control protocol types
- Any tests under `codex-rs/cli/tests/` or nearby CLI command tests

## Implementation steps

1. Create a branch:

   ```bash
   git checkout -b restore-windows-cli-remote-control-0130
   ```

2. Modify `codex-rs/cli/src/remote_control_cmd.rs`:

   - Split `run_foreground_remote_control(...)` into Windows and non-Windows versions.
   - Keep the existing implementation under `#[cfg(not(windows))]`.
   - Add the Windows implementation described above.

3. Keep `handle_remote_control_start(...)` and `handle_remote_control_stop(...)` unchanged.

4. Run formatting:

   ```bash
   cargo fmt --manifest-path codex-rs/Cargo.toml
   ```

5. Run targeted tests:

   ```bash
   cargo test -p codex-cli --manifest-path codex-rs/Cargo.toml remote_control
   ```

6. Run a broader CLI check if feasible:

   ```bash
   cargo test -p codex-cli --manifest-path codex-rs/Cargo.toml
   ```

7. Build on Windows:

   ```powershell
   cargo build -p codex-cli --manifest-path codex-rs/Cargo.toml
   ```

8. Manual Windows validation:

   ```powershell
   codex remote-control
   ```

   Expected result:

   - No Unix daemon lifecycle error.
   - Process remains in foreground.
   - Terminal can stop it with `Ctrl-C`.
   - ChatGPT mobile can discover/select the machine.

## Acceptance criteria

- `codex remote-control` works on Windows native CLI without daemon lifecycle errors.
- `codex remote-control` behavior on macOS/Linux is unchanged.
- `codex remote-control start/stop` behavior is unchanged.
- No changes to authentication, enrollment, pairing, or grant semantics.
- No change to Codex App remote-connection support.
- Code compiles on Windows.
- Existing remote-control tests still pass on supported platforms.

## Risks

### 1. JSON output mode

The current non-Windows implementation can report structured readiness/status information. The Windows compatibility path may not have equivalent readiness data without the daemon helper.

Mitigation:

- Keep `--json` silent/minimal on Windows initially.
- Add structured Windows status only after confirming the runtime exposes stable status without daemon APIs.

### 2. App-server runtime semantics changed since 0.130.0

The old behavior may not map perfectly to the current remote-control startup mode.

Mitigation:

- Use `RemoteControlStartupMode::EnabledEphemeral`, not the removed `remote_control_enabled: true` field.
- Validate against ChatGPT mobile after startup.

### 3. Mobile enrollment may depend on newer app-server RPCs

Newer releases added remote-control pairing/grant RPCs. The foreground compatibility path should not bypass them.

Mitigation:

- Let app-server runtime own enrollment and pairing.
- Avoid manually calling internal RPCs.

### 4. Windows terminal shutdown behavior

Foreground app-server shutdown must remain clean under `Ctrl-C`.

Mitigation:

- Preserve `install_shutdown_signal_handler: true`.
- Test repeated start/stop cycles.

## Future improvement

If this compatibility path proves stable, the cleaner upstreamable fix would be:

- Keep the daemon-backed path for Unix.
- Add a first-class `ForegroundNoDaemon` remote-control host mode for Windows.
- Add Windows-specific tests that assert bare `codex remote-control` does not call daemon lifecycle helpers.
- Add a clear CLI warning that `start/stop` daemon mode is Unix-only, while bare foreground mode is supported on Windows.

## References

- Codex changelog: https://developers.openai.com/codex/changelog
- Remote connections docs: https://developers.openai.com/codex/remote-connections
- Upstream repository: https://github.com/openai/codex
