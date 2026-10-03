---
id: sabado-63-a-recovery-word-is-a-leak-only-in-a-value-bb87e7
type: bug
status: queued
priority: 1
created: 2026-10-03
depends: []
attempts: []
---
# test(e2e): a recovery word is a leak only where it is a value, so journey 5 stops failing by chance

## The bug

Journey 5 (`frontend/e2e/journeys/05-vault.spec.ts`, step « nothing readable left the browser »)
fails on PRs that did not touch the vault. The failures look like:

```
- Array []
+ Array [
+   "POST http://localhost:4173/api/vault/setup carries the recovery word « check »",
+ ]
```

It is a **false positive** in the detector, not a leak. `findLeaks` in `frontend/e2e/fixtures/leak.ts`
matches each of the 12 recovery words with
`wholeToken = (?<![A-Za-z0-9])word(?![A-Za-z0-9])`. It matches against
`${req.url}\n${jsonBody}`. That text includes **URL path segments and JSON keys**, and `_` and
`/` count as boundaries:

- `/api/vault/setup` contains the BIP39 words `vault` and `setup`.
- the `VaultSetupRequest` keys `salt_b64`, `vault_check_b64` and `recovery_phrase_hash` contain
  `salt`, `vault`, `check` and `phrase`.

The phrase is 12 words drawn from the 2048-word list (`@scure/bip39`, `lib/crypto.js:167`).
Five of those words collide, so about **2.9 %** of runs fail at random. CI logs from 2026-09-28 to
2026-10-03 show the words « vault » (×22), « setup » (×10) and « check » (×1) reported this way.

## Requirements

- [ ] **R1** In the readable text, a recovery word counts as a leak only inside a JSON **value**.
      Walk the parsed body recursively and test only string values. Never test keys, never test
      the URL path, and test the URL query string as values only.
- [ ] **R2** Do not weaken the detector anywhere else. `leakForms` (the verbatim and base64
      secrets) still scans the whole wire: method, URL, headers and body. This is the security
      test. A value that carries a word is still a leak.
- [ ] **R3** Red first, in `frontend/e2e/bundle/leak-detector.spec.ts`:
  - [ ] a POST to `/api/vault/setup` whose body is `{"salt_b64":"…","vault_check_b64":"…"}`,
        with the phrase words `vault setup check salt phrase …`, returns `[]`. It fails today.
  - [ ] the same request with `{"note":"my vault word"}` still reports « vault » (the detector
        can still fail).
  - [ ] a word nested in an array or object value is still reported.

## Verify

- [ ] The `bundle` project's `leak-detector.spec.ts` was red before the fix and is green after.
- [ ] Journey 5 is green.
