# Design

## Accounts

### Registry (PDA, seeds: ["registry", mint])
- `mint: Pubkey`
- `admin: Pubkey` (can change attester)
- `attester: Pubkey` (can add, renew, revoke entries)
- `bump: u8`

### Entry (PDA, seeds: ["entry", mint, wallet])
- `registry: Pubkey`
- `wallet: Pubkey`
- `expires_at: i64` (unix seconds)
- `revoked: bool`
- `bump: u8`

An entry is valid when it exists, `revoked == false`, and `Clock.unix_timestamp < expires_at`.

## Instructions, allowlist program

- `init_registry(mint)`: admin signs, creates Registry, sets attester = admin.
- `set_attester(new_attester)`: admin signs.
- `add_entry(wallet, expires_at)`: attester signs, creates Entry.
- `renew_entry(wallet, expires_at)`: attester signs, updates expiry, clears revoked.
- `revoke_entry(wallet)`: attester signs, sets revoked = true.

Unauthorized paths to test: non-admin set_attester; non-attester add, renew, revoke; add with expires_at in the past.

## Instructions, transfer_hook program

- `initialize_extra_account_meta_list(mint)`: writes the ExtraAccountMetaList PDA so Token-2022 passes the two Entry PDAs (derived from source owner and destination owner) and the Clock sysvar on every transfer.
- `execute(amount)`: called by Token-2022 during transfer. Loads both entries, checks validity, errors on failure. Never writes.

Constraint: the hook can only read accounts Token-2022 hands it, and those must be derivable from the transfer's own accounts. That is why entries are keyed by wallet (the token account owner), not by token account.

## Mint

Token-2022 with extensions: TransferHook (program = transfer_hook), MetadataPointer + TokenMetadata (name, symbol). Freeze authority retained as an emergency brake. Mint authority held by the issuer.

## Perihelion application

Perihelion does not run a stake pool and never custodies client stake. An institution's custodian opens native stake accounts delegated across the Tier 1 members. Native stake accounts are not tokens, so the transfer hook has no role in Perihelion. What carries over is the registry; see Operator registry below.

## Operator registry (same program, different attester)

A second Registry instance where the attester is the auditor and entries are operators: "controls reviewed, valid until date." This is the on-chain half of OOAS. Entry semantics are identical.

## Security notes

- Every instruction checks the signer against the stored admin or attester; never trust a passed-in pubkey.
- PDA seeds include the mint so registries for different tokens cannot collide.
- expires_at is compared against Clock, never against a caller-supplied time.
- Mainnet upgrade authority must move from the dev key to a hardened key before any real mint; decide then whether to make the programs immutable.

## Stretch

ConfidentialTransferMint on the same mint, so balances are private while transfers stay gated. Verify the ZK ElGamal proof program is live on the target cluster first.

## Open questions

- Hook access to the Clock sysvar via ExtraAccountMetaList: confirm during the transfer-hook example walkthrough.
- Entry account rent: who pays, and whether revoked entries are closed to reclaim it.
