---
id: cqc-be-37-the-go1-gateway-certificate-is-verified-24925a
type: bug
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# The Go1 gateway's certificate is verified; `rejectUnauthorized: false` goes

Sources: BE-22 (`docs/backend/decisions-2026-09-26.md:49`), `docs/backend/todo.md:55`.

## The bug

`backend/src/projector/ports/go1-gateway-agent.ts:6-12` sets `ca: credentials.caCert` and
`rejectUnauthorized: false // TODO(BE-22)`. Both consumers skip server verification:

- the projector (`projector/ports/index.ts:56`);
- the trigger preflight (`run-lifecycle/ports/index.ts:108-111`).

## Evidence (openssl, 2026-09-30)

Both gateways present public Let's Encrypt chains and return `Verify return code: 0 (ok)`,
valid to 2026-11-12:

- `gateway.qa.go1.cloud`: `CN=go1.cloud`, SAN includes `*.qa.go1.cloud`.
- `gateway.go1.com`: SAN `*.go1.com`.

Node's `ca` option **replaces** the default trust store, which is likely why verification
"had" to be off: `go1-ca-cert` is Go1's client-side CA, not the server's.

## Fix

- Verification on.
- `ca` either removed or `[...tls.rootCertificates, caCert]`.
- The client certificate and key stay.

## Red first (Art. I)

- `go1-gateway-agent.test.ts:14-19`: `rejectUnauthorized` is not `false`, and `ca` is
  `undefined` or contains `tls.rootCertificates`.
- `test/go1-gateway-security.test.ts:8-20` expects **0** occurrences of the TODO.
- Optionally: a local TLS server with a self-signed certificate must be rejected.

## Acceptance criteria

- [ ] Verification is on for both consumers, with the tests above.
- [ ] **Operator, after deploy:** one projector run and one trigger preflight on dev succeed
      against QA. The PR lists this as the merge gate.
- [ ] The PR body says TLS has the same pattern (`resolve-go1-auth.ts:13-18` in
      `translated-language-service`), for the TLS owner.

## Blocked by

- (nothing)
