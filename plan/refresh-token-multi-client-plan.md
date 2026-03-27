# Refresh Token Safety Plan (Repaired)

## 1) Scope and Goal

This plan hardens `codex-switch` for environments where multiple clients may read and write the same `auth.json`.

Primary goal:
- prevent local token-chain downgrade and blind overwrite.

Explicit non-goal:
- guarantee global safety against non-cooperative writers that bypass the protocol.

Secondary goal:
- contain damage from non-cooperative writers via deterministic conflict handling.

## 2) Terms

- `local chain head`: refresh token state currently stored in local `auth.json`.
- `authoritative chain head`: refresh token state accepted by the auth server.
- `chain integrity`: local chain head corresponds to a server-issued token that is not known to be consumed or invalidated, and is backed by an authoritative server outcome within verification window `T` (or is explicitly quarantined under ambiguous-failure flow).
- `writer`: any process that can modify `auth.json`.
- `cooperative writer`: writer that follows this lock + CAS protocol.
- `local safety`: any individual cooperative writer never destroys fresher local state by its own actions.
- `cooperative-global safety`: all cooperative writers preserve chain integrity.
- `damage containment`: non-cooperative interference is detected and converted to explicit conflict, not silent corruption.

## 3) Ontology and Constraints

Refresh token behavior is regime-dependent and not a plain config update.
Therefore:
- multi-writer is safe only with strict coordination;
- under non-cooperative or untrusted-backend conditions, race failures are expected (`refresh token already used`).
- filesystem lock guarantees are backend-dependent (local FS vs weak/distributed/NFS semantics).
- lock trust is policy-based, not inferred optimistically at runtime.
- unknown/untrusted backend must fail closed: write-path degrades to read-only safety mode.
- lease and CAS are independent defenses only on trusted backends; on untrusted backends they are correlated failures on the same substrate.

Token rotation regime must be explicit (`codexSwitch.rotationRegime`):
- `strictSingleUse` (default): linear single-use rotation, no grace.
- `gracePeriod`: provider may temporarily accept parent and child tokens; benign forks may auto-resolve.
- `familyInvalidation`: replay may invalidate the entire token family; recovery uses quarantine + mandatory probe and may require forced re-authentication.

## 4) Safety Invariants

1. (Cooperative scope) No blind writes to `auth.json`.
2. (Cooperative scope) Never overwrite a fresher local chain head with an older profile snapshot.
3. (Cooperative scope) Any write requires active lease ownership acquired atomically.
4. (Cooperative scope) Commit requires CAS-gate check against expected chain fingerprint; mismatch handling is regime-dependent.
5. (Cooperative scope) Conflict must abort write and surface remediation.
6. (Universal scope) Server verdict wins: if provider authoritatively rejects refresh (`invalid_grant`/`already used`), local state is marked stale/conflict and cannot be treated as valid.
7. (Universal scope) Ambiguous failures (timeout/5xx/transport errors) must not be treated as authoritative rejection until probe/retry policy is exhausted.

## 5) State and Metadata

Add lock file (same directory as `auth.json`):
- `auth.lock`:
  - `owner_id`
  - `protocol_version`
  - `fencing_token`
  - `lease_expires_at`
  - `updated_at`

Add operation journal:
- `auth.journal` append-only records:
  - `lease_acquired`
  - `refresh_started`
  - `refresh_committed`
  - `conflict`
  - `rollback`
- journal classification: advisory by default (debug/forensics). Do not treat journal alone as a safety proof on untrusted backends.

Add backup ring:
- `auth.json.bak.1..N` (default `N=5`).

## 6) Freshness and Fingerprints

Freshness precedence:
1. `last_refresh` from payload (if valid).
2. `iat` claim (if valid).
3. `mtime` fallback only.

CAS identity:
- hash fingerprint of current refresh token (`sha256(refresh_token)`), never raw token in logs.

## 7) Protocol

### 7.1 Acquire lease
1. Read `auth.lock`.
2. If expired or absent: acquire with new `owner_id` and incremented `fencing_token`.
3. Acquisition must be atomic (`create-exclusive` or equivalent rename-based CAS lock).
4. If active and foreign owner: fail fast in `strict` mode.
5. Lease is allowed only when current backend is trusted by policy (`trustedLockBackend`) and not in probation.
6. If backend is untrusted/unknown or in probation, refuse lease and force read-only safety mode.

### 7.2 Pre-write reconciliation
1. Read disk `auth.json`.
2. Compare identity and freshness against in-memory profile snapshot.
3. Identity match is strict tuple:
   - `(subject OR chatgptUserId OR userId) + accountId + workspaceId(defaultOrganizationId)`.
4. If disk is fresher for same identity: update stored profile first (pull), skip push.

### 7.3 Commit with CAS
1. Capture expected refresh fingerprint from disk before write.
2. Prepare next payload.
3. Re-read disk fingerprint just before commit.
4. If mismatch:
   - Under `strictSingleUse`: `Conflict`, abort write, journal event, user action required.
   - Under `gracePeriod`: attempt auto-resolution only if provider offers non-consuming validity evidence (or explicit grace semantics). Compare freshness using §6 precedence (`last_refresh` -> `iat` -> `mtime`); if disk token is fresher, pull it silently and skip push; if prepared token is fresher, proceed with write. If validity evidence is unavailable or freshness is indeterminate, treat as ambiguous failure and follow §10 ambiguous flow.
   - Under `familyInvalidation`: `Conflict`, abort write, and enter quarantine with mandatory bounded probe; force re-auth only on authoritative rejection or exhausted probe policy.
5. If match: write temp file, backup rotate, atomic replace.
6. If subsequent refresh attempt is authoritatively rejected by the server, transition to `Conflict` and block further writes until reconciliation. (Ambiguous failures are handled by step 7.)
7. Classify refresh failures:
   - `authoritative rejection`: immediate stale/conflict transition.
   - `ambiguous failure`: bounded probe/retry before stale-marking.

### 7.4 Release lease
1. Final journal event.
2. Release lock if owner matches and fencing token unchanged.

## 8) Runtime Policies

Add `codexSwitch.multiClientPolicy`:
- `strict` (default for SSH/remote):
  - one refresh writer only;
  - second writer blocked with explicit message.
- `compat`:
  - old behavior allowed but with warnings;
  - still enforce CAS to avoid silent overwrite.
- repeated conflict guard:
  - after `N` consecutive conflicts (default `N=3`) switch extension to read-only safety mode until operator action.
- `codexSwitch.trustedLockBackend`:
  - explicit allowlist of lock backends where write-path is permitted.
  - default fail-closed for unknown backends.
- lock probation:
  - first lock anomaly (or failed CAS under held lease) demotes backend/session to probation read-only mode until explicit operator re-enable.
- probation auto-recovery:
  - after `M` consecutive clean checks (default `M=5`) on a policy-trusted backend that was demoted to probation due to anomaly, backend/session exits probation automatically (policy can disable auto-recovery via `codexSwitch.probationAutoRecovery: false`).
  - `clean check` definition: successful lock self-test roundtrip (acquire + release test lock), no lease/CAS anomalies, and no authoritative provider rejection during the check interval.

## 9) File-Level Implementation Plan

- `src/auth/auth-manager.ts`
  - parse freshness signals and compute token fingerprint helper.
- `src/auth/codex-auth-sync.ts`
  - guarded write path:
    - lease validation
    - CAS gate
    - backup ring
    - journal hooks
- `src/auth/profile-manager.ts`
  - lock lifecycle, reconciliation logic, watcher-triggered pull sync.
  - map provider rejection signals (`invalid_grant`, `already used`) to `Conflict` state and trigger re-auth recovery path.
  - ambiguous failure classifier + bounded probe/retry policy before stale-marking.
- `src/commands/index.ts`
  - conflict UX and recovery actions:
    - re-import current auth
    - switch to last known valid profile
- `README.md` + `CHANGELOG.md`
  - document cooperative safety model, damage containment semantics, and non-goals.

## 10) Recovery Flows

When conflict detected:
1. stop write;
2. branch on rotation regime:
   - `strictSingleUse`: import current `auth.json` into active profile; re-attempt switch under lease.
   - `gracePeriod`: attempt auto-resolution only with non-consuming validity evidence and §6 freshness precedence (`last_refresh` -> `iat` -> `mtime`); if auto-resolution succeeds, resume without operator action; otherwise escalate to operator conflict.
   - `familyInvalidation`: enter quarantine and run mandatory bounded probe; if provider authoritatively rejects (or probe policy is exhausted), force re-login/re-auth.
3. if conflicts repeat above threshold: enter read-only safety mode and require explicit operator confirmation to resume writes.
4. if server authoritatively rejects refresh (`invalid_grant`/`already used`): treat local chain as stale, force re-login/import before resuming writes.

When server response is ambiguous:
1. hold write-path for affected profile/session;
2. run bounded probe/retry (`K` attempts, exponential backoff);
3. if unresolved, escalate to `Conflict` with explicit operator path.

When parse/corruption detected:
1. restore latest valid backup from ring;
2. journal rollback;
3. prompt user to re-login if needed.

## 11) Test Matrix

Unit tests:
- freshness ordering;
- identity matching;
- CAS success/failure branches;
- lock acquire/expire/fencing behavior.

Integration tests:
- two cooperative clients, repeated race loops;
- stale writer blocked by fencing token;
- backup restore after forced corruption;
- `strict` policy blocks concurrent writer.
- non-cooperative writer modifies `auth.json` during lease window -> deterministic Conflict, no silent overwrite.
- clock-skew scenarios across clients -> lease + fencing remains safe.
- local CAS passes but server rejects refresh -> stale/conflict transition and mandatory re-auth flow.
- weak-lock backend simulation (NFS-like semantics) -> lease refusal and automatic read-only safety mode.
- false-negative reliability probe simulation -> backend remains fail-closed unless explicitly trusted by policy.
- ambiguous server failure (timeout/5xx) -> bounded probe path before stale-marking.
- grace-period regime with validity evidence -> benign fork auto-resolves without operator conflict.
- grace-period regime without validity evidence -> falls to §10 ambiguous flow, not silent auto-resolve.
- family-invalidation regime -> quarantine + mandatory probe path is triggered; forced re-auth only after authoritative reject or exhausted probe policy.

Acceptance criteria:
- no silent downgrade of local chain head;
- no blind overwrite on race;
- deterministic conflict path with explicit operator action.
- claims are explicitly scoped to cooperative writers; non-cooperative behavior is contained, not guaranteed-safe.
- no accepted post-conflict state after provider rejection; authoritative incompatibility always forces conflict + re-auth/import flow.
- extension automatically disables write-path (read-only safety mode) when lock reliability is not established, including unknown/untrusted backends unless explicitly trusted by `codexSwitch.trustedLockBackend`.
- ambiguous server failures do not immediately force stale-marking; bounded probe/retry is enforced first.
