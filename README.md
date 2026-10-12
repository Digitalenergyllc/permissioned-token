# Permissioned Token

A Token-2022 mint whose transfers are gated by an on-chain allowlist, enforced by a transfer hook.

**Status:** design. Turbin3 Q4 Builders 2026 capstone. Nothing here is deployed to mainnet.

## Problem

Institutions cannot hold a token that anyone can send to anyone. Their rules require that every holder be vetted, that vetting expire, and that it be revocable. Today that is enforced off-chain, in agreements and spreadsheets, which means it is not enforced at the point of transfer.

## What this is

A token that carries its own allowlist. An attester approves wallets with an expiry and can revoke them. A transfer hook checks sender and receiver on every transfer and rejects any that are not current. The chain enforces the rule.

## Two programs

- `allowlist`: the registry. One `Registry` per mint (admin, attester). One `Entry` per wallet (expires_at, revoked). Only the attester can add, renew or revoke.
- `transfer_hook`: implements the SPL transfer-hook interface. On every transfer it reads both entries and the Clock and allows or rejects. Read-only.

Design detail, including the stake-pool wrapper and the fallback, is in [docs/design.md](docs/design.md).

## Demo

Mint to an approved wallet. Transfer to a second approved wallet succeeds. Transfer to an unapproved wallet fails. Revoke the second wallet, retry, fails. Advance the clock past expiry, retry, fails.

## Why a hook and not default-frozen accounts

Token-2022's `DefaultAccountState` lets an issuer freeze new accounts and thaw them after KYC. That checks once, at account creation. A hook checks every transfer, so expiry and revocation take effect immediately. The frozen-account pattern is the documented fallback where a hook cannot be used.

## Prior art

- [Operator Register](https://app.extol.work/groups/node-operator-register): validator conduct attestations with oracle evidence. Different subject (slot conduct, not holder eligibility). This project does not cover conduct.
- [OOAS](https://github.com/Digitalenergyllc/ooas-spec): the off-chain attestation format whose control IDs and attester role this registry mirrors on-chain.
- Token-2022 transfer-hook interface and `DefaultAccountState` extension, Solana Program Library.

## License

MIT.
