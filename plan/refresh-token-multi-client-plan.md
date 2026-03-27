# Refresh Token Safety Plan (Repaired)

## 1) Scope and Goal

This plan hardens `codex-switch` for environments where multiple clients may read and write the same `auth.json`.

Primary goal:
- prevent local token-chain downgrade and blind overwrite.

Explicit non-goal:
- guarantee global safety against non-cooperative writers that bypass the protocol.

## 2) Terms

- `chain head`: the currently valid refresh token state.
- `writer`: any process that can modify `auth.json`.
- `cooperative writer`: writer that follows this lock + CAS protocol.
- `local safety`: extension never destroys fresher state by its own actions.
- `global safety`: all writers preserve chain integrity.

## 3) Ontology and Constraints

Refresh token rotation is a linear, one-time state transition, not a plain config update.
Therefore:
- multi-writer is safe only with strict coordination;
- without coordination, race failures are expected (`refresh token already used`).

## 4) Safety Invariants

1. No blind writes to `auth.json`.
2. Never overwrite a fresher chain head with an older profile snapshot.
3. Any write requires active lease ownership.
4. Commit requires CAS check against expected chain fingerprint.
5. Conflict must abort write and surface remediation.

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
3. If active and foreign owner: fail fast in `strict` mode.

### 7.2 Pre-write reconciliation
1. Read disk `auth.json`.
2. Compare identity and freshness against in-memory profile snapshot.
3. If disk is fresher for same identity: update stored profile first (pull), skip push.

### 7.3 Commit with CAS
1. Capture expected refresh fingerprint from disk before write.
2. Prepare next payload.
3. Re-read disk fingerprint just before commit.
4. If mismatch: `Conflict`, abort write, journal event, user action required.
5. If match: write temp file, backup rotate, atomic replace.

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
- `src/commands/index.ts`
  - conflict UX and recovery actions:
    - re-import current auth
    - switch to last known valid profile
- `README.md` + `CHANGELOG.md`
  - document cooperative safety model and non-goals.

## 10) Recovery Flows

When conflict detected:
1. stop write;
2. import current `auth.json` into active profile;
3. re-attempt switch under lease.

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

Acceptance criteria:
- no silent downgrade of chain head;
- no blind overwrite on race;
- deterministic conflict path with explicit operator action.
