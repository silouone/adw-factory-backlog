---
id: cqc-be-41-cqc-has-its-own-go1-identity-9eddef
type: manual
status: queued
priority: 2
created: 2026-09-30
depends: []
attempts: []
---
# CQC calls Go1 as itself, with rotated secrets

**Operator (Silou → Go1).** Sources: BE-20 (`decisions-2026-09-26.md:48`), BE-26 (:75),
`todo.md:14, 20, 61-62`.

## Why

- CQC uses credentials copied from TLS, so Go1 sees our calls as TLS.
- The copied root JWT can **write** (`/set`).
- The Go1 client values and the admin tokens pasted on 2026-09-10 must be rotated before prod
  (BE-26).

## Steps

- Ask Go1 for a CQC client certificate, key and CA, and, if possible, a **read-only** JWT.
- Write them to `/credentials/{stage}/content-quality-checker/go1-{root-jwt,client-cert,client-key,ca-cert}`.
  The names don't change, so no code change is needed
  (`projector/ports/get-go1-credentials.ts:12-29`).
- Use SecureString with the default `aws/ssm` key. The stack has no `kms:Decrypt`, so a custom
  key would break reads.
- Rotate the TLS-sourced values CQC still uses, and the 2026-09-10 admin tokens.
- If Go1 issues a read-only token, update the wording of BE-20.
- Co-schedule with `cqc-be-37`: a new identity may come with a new CA.
- Noticed, not ours: TLS stores prod and staging `go1-client-*` as plain `String`. Tell the
  TLS owner.

## Done when

- [ ] Dev projector and trigger preflight run on the CQC identity.
- [ ] The rotation is recorded here with its date.

## Blocked by

- (nothing)
