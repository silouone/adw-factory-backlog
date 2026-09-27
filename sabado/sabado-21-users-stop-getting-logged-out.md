---
id: sabado-21-users-stop-getting-logged-out
type: feat
status: done
priority: 3
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side, sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account]
attempts: [{"runId":"sabado-21-users-stop-getting-logged-out-1789905459432","branch":"adw/sabado-21-users-stop-getting-logged-out","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-21-users-stop-getting-logged-out-1789905459432/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1230"}]
---
# fix(auth): one refresh-token minting path, no cross-tab logout, and the coffre locks on inactivity

> **Audit:** **P1-2**, **P1-3** (axis D), the coffre timer (D P2) and the
> Redis-down 500 on Google sign-in (half of **P1-14**) — Lane F.
> Depends on `sabado-11` because both edit `backend/app/auth/router.py`
> and `test_google.py`; this one goes second.

## What happens today

- **P1-2.** `router.py:879` `create_refresh_token(user.id, user.email)` —
  default `version=0` (`jwt.py:50`) — while `/auth/refresh` refuses
  `ver != user.sessions_version` (`router.py:676`). The #879 change touched
  `_issue_tokens` only. Any account that reset or changed its password
  (`sessions_version ≥ 1`) and then uses « Continuer avec Google » gets 401
  at `/auth/refresh`; `AuthCallbackPage.tsx:32` sends it to `/login`; loop.
  Both features shipped in September.
- **P1-3.** `frontend/src/contexts/AuthContext.tsx:187-215`: a 60-minute
  idle timer (`IDLE_LOGOUT_MS`, `:69`) re-armed on `pointerdown`,
  `keydown`, `visibilitychange` — never again in a hidden tab. Its logout
  calls `/auth/logout`, which blocklists the **shared** refresh cookie
  (`router.py:694-697`); `AuthContext.tsx:102-104` clears the session on
  any refresh failure. `grep BroadcastChannel|navigator.locks|'storage'`
  → 0. Two tabs (a document opened "in a new tab", the PWA plus a browser
  tab): the background tab signs the working tab out within 60 minutes;
  two tabs refreshing together reproduce #751 across tabs.
- **Coffre timer.** "15 min of inactivity" is in fact "15 min after
  unlock": `VaultContext.jsx:41` keeps its own `timerRef` / `resetTimer`
  (`:57`) while `stores/vaultCountdown.ts` counts down independently — two
  timers, and the store's is the one that locks.
- **Redis down.** `router.py:850` `get_redis().setex(f"oauth_state:…")` is
  unguarded; with Redis down Google sign-in 500s (incident #6,
  2026-08-06). `redis_store.py:35,42,60` are fail-open by design; this
  site is the one that is not, and it fails the wrong way.

## Requirements

- [ ] **R1** `router.py:879` mints through `_issue_tokens`, so
      `create_refresh_token(` has exactly one call site (`_issue_tokens`).
- [ ] **R2** Idle logout consults a shared `localStorage` stamp
      (`sabado.lastActivityAt`, written on every activity event by every
      tab) before logging out; a tab whose stamp is fresh does not log out.
- [ ] **R3** The refresh is serialised across tabs with
      `navigator.locks.request('sabado-refresh', …)` (Web Locks — in every
      supported browser); inside the lock, a tab re-checks whether another
      tab already refreshed. In `client.ts` only if the single-flight logic
      lives there.
- [ ] **R4** One owner for the coffre countdown: `vaultCountdown` gains an
      `extend()` action; `VaultContext.resetTimer` calls it; `timerRef` is
      deleted.
- [ ] **R5** The OAuth `setex` at `router.py:850` is guarded: Redis
      unreachable → `503` with a French detail, never a 500.

## Files

`backend/app/auth/router.py` (line 879 region and the `setex` at 850;
**not** `_login_or_link`, which `sabado-11` changed) ·
`backend/tests/test_google.py` · `frontend/src/contexts/AuthContext.tsx` ·
`frontend/src/api/client.ts` (only under R3) ·
`frontend/src/contexts/AuthContext.idle.test.tsx` ·
`frontend/src/contexts/AuthContext.test.tsx` ·
`frontend/src/stores/vaultCountdown.ts` ·
`frontend/src/contexts/VaultContext.jsx` ·
`frontend/src/stores/vaultCountdown.test.ts` ·
`frontend/src/contexts/VaultContext.test.jsx`.

## Verify

- [ ] Red test: `test_google.py::test_google_login_after_a_password_change_refreshes`
      — change the password (bumps `sessions_version`), sign in with
      Google, call `/auth/refresh` with the cookie → 200. RED today (401).
- [ ] Red test: `test_google.py::test_google_start_answers_503_when_redis_is_down`
      (monkeypatch `get_redis` to raise) — RED today (500).
- [ ] Red test: `AuthContext.idle.test.tsx` "another tab was active — no
      logout": stamp `sabado.lastActivityAt` fresh from outside the
      provider, advance 60 min, assert no `/auth/logout` call. RED today.
- [ ] Red test: two `AuthProvider`s sharing one mocked cookie, both hit a
      401 at once → exactly one `/auth/refresh` request. RED today.
- [ ] Red test through both modules: unlock, 14 min, `pointerdown`, 2 min
      → still unlocked; 15 more min idle → locked. RED today.
- [ ] `grep -rn "create_refresh_token(" backend/app | grep -v _issue_tokens
      | grep -v "def create_refresh_token"` → nothing.
- [ ] `test_auth.py` (rotation, revocation, "signs the other devices out
      but not this one"), `AuthContext.test.tsx` (single-flight),
      `VaultContext.test.jsx` (nothing in storage after unlock) — green
      unchanged.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.

## Out of scope

The `/health` verdict (`sabado-22`); `Secure` cookie on staging (D P3);
server-side reuse grace on a rotated `jti`; `drive.readonly` scope.
