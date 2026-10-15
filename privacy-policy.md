---
title: Privacy Policy
---

# Manas — Privacy Policy

**Version 1.1 — 2026-09-09.**

## 1. Who we are & scope

Manas is a private, single-household software platform. It is built and run by one individual (the owner) on
one server the owner physically owns and operates in the owner's home; there is no company, no employees, and
no public sign-up. This policy covers the owner's own household
members who use Manas ("you," "end users") — it is not a public product, and no one outside the household is a
user. Plaid Inc. is the only third-party service that processes your financial data on Manas's behalf.

## 2. What data we collect

**Via Plaid.** When you connect a bank account through Plaid Link, Manas requests these Plaid products: Auth,
Transactions, Recurring Transactions, and Liabilities. Depending on the product and what your institution
returns, this can include: account and routing identifiers, account balances, transaction history, liability
details (loan/card balances, APR, minimum/due amounts), and recurring payment streams. Manas's connector is
provisioned read-only by default — a `read` grant, never a money-movement `act` grant, unless a separate grant
is explicitly turned on.

**Collected directly by Manas.** Your login credential — a password (adults) or numeric PIN (children) — is
never stored in the clear; it is hashed (Argon2id) before it is checked or stored. Security-relevant actions
(logins, Plaid token reads, connection changes) generate audit log entries recording only an outcome, an id,
and a timestamp — never a password, a token, or account content.

## 3. How we use it

Data collected through Manas is used only to show your household's own finances back to you — balances,
transactions, trends, and bill/budget advice inside the household's own apps (Artha) — and to secure and audit
access to your account. It is never used for advertising, never profiled or sold, and never shared with anyone
outside Plaid (§4) and the household itself.

## 4. Plaid

Connecting a bank account uses Plaid Link. By completing that flow you authorize Plaid to collect, use, and
share the resulting financial data with Manas, subject to Plaid's own end-user privacy policy:
https://plaid.com/legal/#end-user-privacy-policy. Manas connects to Plaid's **Production** environment for the
household's own accounts, read-only.

You can review and revoke Manas's access to your accounts directly with Plaid through the Plaid Portal at
https://my.plaid.com, in addition to revoking the connection inside Manas itself (§6).

## 5. Storage & security

Plaid access tokens are envelope-encrypted per connection before they ever reach the database; no public
interface ever returns a raw token, and the database that holds account/transaction/liability data is a local
PostgreSQL instance on the owner's own server — never a public cloud. The encryption key protecting all of this
is unsealed only by an adult logging in (or the recovery code); it cannot be decrypted without that.
Full technical detail is in the [Security Policy](security-policy.md) §2–§4.

## 6. Retention & deletion

Revoking a bank connection crypto-shreds its access token and blanks the stored ciphertext immediately — this
is enforced in code, not a manual step.

Synced account and transaction data is kept under a configured retention schedule: records are held for 2 years
as current data and a further 5 years as history, then automatically tombstoned by a periodic sweeper (7 years in
total). To have your account/transaction data deleted sooner, ask the household owner directly; on that request
the data is deleted from the database. A self-service delete action in the app is planned.

## 7. Children

Household members with the Child role never see or touch bank connections. Every route that starts or
completes a Plaid Link connection refuses a child actor before doing anything else, deterministically, in code.
Separately, a household-finance source consent can never be owned by, or acted on by, a child — checked before
any positive adult check runs, so a mis-provisioned account is still refused. A child can see money data only if
an adult explicitly shares a view with them; a child can never be granted that access, only receive it from an
adult who already holds it.

## 8. Your rights & contact

You can ask to see, correct, or delete the data Manas holds about you, or ask questions about this policy, by
contacting the household owner directly — in person, or by the household owner's email.
Manas has no separate support desk or company; the household owner is the sole contact and sole administrator.

## 9. Changes

This policy is versioned at the top of this document. Material changes to what data Manas collects, how it
uses Plaid, or how it stores/retains your data will be reflected here with a new version and date, reviewed by
the owner at least annually alongside the [Security Policy](security-policy.md).
