# OpenClaw Repo Review (AI Architecture + AI App Design)
Date: 2026-02-02

## Findings (Most Critical First)
1. Critical: Plugin install runs arbitrary lifecycle scripts.
   - npm install is invoked without --ignore-scripts, so any plugin dependency can execute code at install time with full host privileges. This is a supply-chain RCE vector, especially since plugins can be fetched from npm specs.
   - Mitigation: default to --ignore-scripts, introduce an explicit opt-in flag for scripts, or run installs inside a locked-down sandbox/containers.
   - Reference: src/plugins/install.ts:216

2. Critical: Plugin execution is fully trusted and runs in-process with powerful host APIs.
   - Plugins are loaded via jiti(...) and executed directly in the gateway process, and the runtime API exposes runCommandWithTimeout and writeConfigFile, enabling arbitrary command execution and config mutation without the exec approval model. If plugins are not strictly trusted, this breaks the security boundary.
   - Mitigation: add a capability-based permission model, isolate plugins in a worker/sandbox, and narrow the runtime surface.
   - References: src/plugins/loader.ts:292, src/plugins/runtime/index.ts:165

3. High: Tar extraction lacks explicit path-traversal/symlink defenses.
   - ZIP entries are validated, but tar extraction uses tar.x without filtering entry paths or link types. A crafted tarball could potentially write outside destDir or exploit symlinks if the tar library allows it.
   - Mitigation: use onentry/filter with explicit path normalization and reject symlinks/hardlinks; consider strip and explicit allowlist of entry types.
   - Reference: src/infra/archive.ts:109

4. Medium: runCommandWithTimeout does not terminate child process trees.
   - On timeout it sends SIGKILL only to the direct child; grand-children can live on. This can leak resources, bypass timeouts, and keep untrusted code running.
   - Mitigation: spawn in a new process group and kill the group, or use a cross-platform kill-tree helper.
   - Reference: src/process/exec.ts:113

5. Medium: Plugin cache can leave gateways running stale plugin code.
   - The cache key only uses config + workspace, so plugin updates on disk won’t take effect until restart or cache bypass.
   - Mitigation: add plugin manifest mtime/hash into cache key or allow runtime reload to disable cache.
   - References: src/plugins/loader.ts:167, src/gateway/server-plugins.ts:1

6. Low: Temp dirs from plugin install are never cleaned.
   - mkdtemp(...) is used for archive extraction and npm pack, but there is no cleanup. This leaves artifacts on disk and can accumulate.
   - Mitigation: fs.rm(tmpDir, { recursive: true, force: true }) in finally.
   - References: src/plugins/install.ts:272, src/plugins/install.ts:411

7. Low: Async plugin registration is ignored.
   - If register() returns a promise it is ignored (only a warning is logged), so async setup can race and leave partially initialized plugins.
   - Mitigation: either await async registers or disallow/throw on async registration to make plugin behavior deterministic.
   - Reference: src/plugins/loader.ts:409

## Open Questions / Assumptions
1. Are plugins considered fully trusted local code? If yes, document that explicitly and surface a warning in the install UX; if no, the current model needs sandboxing and a permissions system.
2. Do you intend plugin installs to be safe for untrusted npm specs? If yes, the install flow should block scripts by default and validate archives defensively (tar path checks).

## Test Gaps / Suggested Tests
1. Add a tar-path traversal test (e.g., ../evil) and a symlink test to verify extraction can’t escape.
2. Add tests around plugin update/reload behavior to ensure code changes are actually picked up.
3. Add an install test that asserts lifecycle scripts are blocked or require explicit opt-in.

## Notes
- Tests not run (review only).
