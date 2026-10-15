---
title: Security Policy
---

# Manas — Information Security Policy

**Version 1.2 — 2026-09-09** (1.1 was 2026-09-03, retention-sweeper correction; this version aligns §2/§10 with
Production Plaid connectivity and generalizes a few operational specifics for the public copy of this document).
Reviewed by the owner annually or on any material change.

## 1. Scope & roles

Manas is a household platform built and run by a single individual (the owner), who is also its sole developer,
administrator, and security contact. It runs on one server the owner physically owns. There are no employees and
no contractors. End users are members of the owner's own household; they hold no administrative access to
infrastructure or code. Plaid is the only third-party financial-data processor in use.

## 2. Data handled

Manas stores Plaid access tokens and the account/transaction/liability data Plaid returns, inside a service
called Sangam. Each token is encrypted before it reaches the database: it is encrypted under a fresh,
per-connection sub-key before any write, and no method on the vault's public interface ever returns raw token
bytes again — reads return an opaque reference, never the secret.
The only code path that ever reconstructs a plaintext token is an internal-only resolver, not part of any public
interface — enforced by an automated architecture test.

Encryption is envelope encryption with per-record sub-keys wrapped under a tier data-encryption key (DEK); the
sub-keys are held in a durable store (SQLite in development, PostgreSQL in production) — the store actually
wired into the production composition, not the in-memory store the codebase also keeps for tests.
That DEK is itself unsealed at login, wrapped under a key derived from each adult's login credential (Argon2)
plus one recovery wrapper, so finance/token data cannot be decrypted without an adult login or the recovery code.
Revoking a connection crypto-shreds its sub-key and blanks the stored ciphertext immediately.
Manas connects to Plaid's **Production** environment for the household's own accounts, read-only.

## 3. Hosting

Manas runs on a single on-premises Linux server in the owner's home; no financial data is stored in any public
cloud. The supporting AI/voice engines (speech-to-text, text-to-speech, local model server) bind exclusively to
`127.0.0.1` and are never exposed to the network. We found no use of a disabled/bypassed TLS
certificate-validation callback anywhere in the codebase (the usual way such a check is silently defeated).
*(Further operational detail about the current network-exposure posture of the public-facing endpoint is
omitted from this public copy.)*

## 4. Access control

Members authenticate with a password (adults) or numeric PIN (children), verified with a constant-time check
plus a decoy hash on any miss, so a failure never reveals whether an account exists.
A password/PIN login mints only a session-grade token — the mint's own assurance guard refuses "strong" from
either factor. Strong assurance requires a WebAuthn/FIDO2 passkey or device Face ID, and this is implemented
today, not merely planned.
Adult and child are distinct roles, and the finance module enforces its own strong-assurance floor in code
before any export or standing-grant action, built so it can only be satisfied by a verified strong-factor human
login — never forged by a caller or an automated principal.

## 5. Logging & monitoring

Manas keeps an append-only, hash-chained audit ledger (Saakshi) for security events. The event recording a Plaid
token read carries only an id and a short fixed reason string — never the token, never account content.
Login attempts are likewise recorded content-free: outcome, member id, timestamp — never a secret or transcript.

## 6. Secure development

Every commit is checked by an automated secret-scanning gate (regex + entropy over the tracked tree) run in CI,
which self-tests the scanner with a planted positive before trusting a clean result. All code is authored and
reviewed by the owner; full history is kept in git. Formal multi-person review does not apply — there is one
developer.

## 7. Secrets management

Real per-install secrets (database password, API keys, the Plaid client secret) live in a single environment
file outside the git repository, delivered to each Manas service unit via systemd's `EnvironmentFile=`
mechanism. That file is restricted to mode `0640`, owned by the service account — never world-readable.
`.gitignore` excludes all local environment files, credential files, and sealed-key material, so none reach git
history.

## 8. Incident response

On suspected compromise of a Plaid token or the server, our procedure is: (1) remove the affected Item(s) via
Plaid's `/item/remove` or the Plaid dashboard, revoking Plaid's own access; (2) rotate the Plaid client secret;
(3) re-provision Manas's own encryption keys so prior key material is invalidated;
(4) review the Saakshi audit ledger for the window to determine what was accessed; (5) re-enroll household
credentials if credential compromise is suspected. This is stated policy, not a system exercised against a real
incident.

## 9. Data retention & disposal

Retention is a configured, enforced matrix, not a manual habit. Every finance data class is assigned a Hot
horizon (2 years) and a Warm horizon (a further 5 years); a row older than both is purgeable. The loader
validates the matrix and fails closed on any malformed entry.
A periodic sweeper classifies finance-transaction rows against that matrix and tombstones the purgeable ones;
each sweep reports evaluated/purged/skipped counts.
Consent records are retention-exempt by design — the grant/revoke audit trail is never rewritten or purged.

Disposal of credentials is immediate: removing a bank connection crypto-shreds its per-connection sub-key and
blanks the stored ciphertext, so the Plaid access token is unrecoverable from that moment (§2). Plaid's own access
is ended with `/item/remove`. Deletion of a member's synced account/transaction data ahead of the retention
horizon is performed by the owner on request. **Planned**: a self-service "delete my financial data" action in the
console that runs that deletion without the owner.

## 10. Vendor management

Plaid is the sole financial-data vendor. We do not resell, share, or otherwise disclose Plaid-provided data to
any other party.

## 11. Review cadence

This policy is reviewed by the owner at least annually and after any material change to how Manas handles Plaid
data, hosting, or access control. Version/date are recorded at the top of this document.

---

**Planned (not yet implemented):**
- A hardening pass over the OS-level service sandboxing beyond what is already in place.
- A self-service financial-data deletion action ahead of the retention horizon (§9).
