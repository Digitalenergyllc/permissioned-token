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

Institution deposits into a Perihelion-managed SPL stake pool; the pool delegates across the Tier 1 operators; Perihelion holds the pool manager key only and never custodies stake.

The SPL stake pool program rejects a pool mint with the TransferHook extension (verified from source, `is_extension_supported_for_mint`, Oct 12 2026). Supported pool-mint extensions: Uninitialized, TransferFeeConfig, ConfidentialTransferMint, ConfidentialTransferFeeConfig, DefaultAccountState, InterestBearingConfig, MetadataPointer, TokenMetadata.

### Option A, wrapper (this capstone)
A third program vaults plain pool tokens and mints a 1:1 permissioned Token-2022 token with the hook. Burn to unwrap. The vault is program-owned with no admin withdraw path. Open for counsel: whether a program-owned vault changes the custody analysis.

### Option B, default-frozen pool mint (first shippable fallback)
`DefaultAccountState = Frozen` on the pool mint itself. Perihelion as freeze authority thaws accounts after KYC and refreezes on termination. No wrapper, no vault. Gates accounts rather than transfers; no expiry. Supported by the stake pool program as-is.

Decision: build A in the course; ship B first if a mandate lands before A is hardened; A is the upgrade path.

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
- Whether the wrapper (Option A) should allow partial unwrap.
- Entry account rent: who pays, and whether revoked entries are closed to reclaim it.
