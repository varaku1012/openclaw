# Resilience & Reliability Patterns

OpenClaw is designed for high availability despite flaky external APIs. Follow these patterns when adding new integrations.

## 1. The Auth Profile System
Always use the centralized `AuthProfile` system for AI providers.
- **Failover:** Automatic round-robin rotation across multiple keys.
- **Cooldowns:**
    - Transient Error: 1-60 min exponential backoff.
    - Billing/Quota Error: 5-24 hr backoff.
- **Implementation:** `src/agents/auth-profiles/usage.ts`.

## 2. Integration Best Practices
- **Retry Logic:** Use exponential backoff for transient network errors (ECONNRESET, ETIMEDOUT).
- **Timeouts:** Never allow infinite hangs. Use `fetchWithTimeoutGuarded`.
- **Circuit Breaker:** If a provider fails repeatedly (e.g., 50% failure rate over 5 min), stop trying for a reset period.
- **Idempotency:** Ensure hooks and tool calls can be retried without side effects.

## 3. Media Pipeline Safety
- **SSRF Protection:** Use the `ssrf` module to block requests to private network IPs (127.0.0.1, 10.0.0.0/8, etc.) when fetching user-provided URLs.
- **Graceful Failure:** If image/audio understanding fails, the agent should still receive the text message and a "Media unavailable" notification instead of crashing.
- **Malware Scanning:** Be cautious with binary buffers; use established libraries like `sharp` for images which have built-in safety.

## 4. Storage Reliability
- **Atomic Writes:** Write new session data to a `.tmp` file and then `rename()` it to the final destination. This prevents data corruption on power loss.
- **Locking:** Use `proper-lockfile` to prevent concurrent write access to session files.
- **JSONL Format:** Sessions use JSON-Lines (one JSON object per line) to allow fast appending and partial recovery if a file is truncated.

## 5. Security Invariants
- **Timing Attacks:** Use `safeEqual` for comparing secrets or tokens.
- **Permission Boundary:** Verify the `agentDir` and `workspace` paths are isolated and cannot be escaped via `../`.
- **Credential Masking:** Ensure logging transport redacts keys and tokens before writing to disk.
