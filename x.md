# NUT-28: Proof of Liabilities

`optional`

---

Proof of Liabilities (PoL) enables wallets to verify that a mint has not inflated its token supply using Merkle Sum Trees with Bitcoin-anchored timestamps.

## Types

### `MerkleSumNode`

| Field | Type | Description |
|-------|------|-------------|
| `hash` | hex64 | SHA256 digest (lowercase) |
| `amount` | uint | Sum of subtree amounts |

### `MerkleSumProof`

| Field | Type | Description |
|-------|------|-------------|
| `leaf_value` | string | Leaf identifier (`B_` or `Y`) |
| `leaf_amount` | uint | Amount at leaf |
| `siblings` | `[[hex64, uint, "L"\|"R"], ...]` | Sibling nodes |
| `root_hash` | hex64 | Expected root |
| `root_amount` | uint | Expected total |

### `EpochReport`

| Field | Type | Description |
|-------|------|-------------|
| `keyset_id` | hex | Keyset identifier |
| `epoch_date` | string | `YYYY-MM-DD` (UTC) |
| `epoch_start` | uint | Unix timestamp 00:00:00 UTC |
| `epoch_end` | uint | Unix timestamp 23:59:59 UTC |
| `previous_epoch_hash` | hex64 \| null | Chain link |
| `cumulative_minted` | uint | Total minted since genesis |
| `cumulative_burned` | uint | Total burned since genesis |
| `total_minted` | uint | Minted this epoch |
| `total_burned` | uint | Burned this epoch |
| `outstanding_balance` | uint | `cumulative_minted - cumulative_burned` |
| `mint_root_hash` | hex64 | Mint tree root |
| `mint_root_amount` | uint | Mint tree sum |
| `burn_root_hash` | hex64 | Burn tree root |
| `burn_root_amount` | uint | Burn tree sum |
| `report_timestamp` | uint | Generation time |
| `report_hash` | hex64 | Canonical hash |
| `report_signature` | hex | Schnorr signature over `report_hash` |
| `ots_proof` | base64 \| null | OpenTimestamps proof |
| `ots_confirmed` | bool | Blockchain anchored |
| `expires_at` | uint \| null | Expiration timestamp for pruning |

### `PoLRootsResponse`

Same as `EpochReport` plus:

| Field | Type | Description |
|-------|------|-------------|
| `is_closed` | bool | Epoch finalized |

### `PoLHistoryResponse`

| Field | Type |
|-------|------|
| `keyset_id` | hex |
| `epochs` | `[EpochReport]` |
| `chain_valid` | bool |

### `PoLVerifyResponse`

| Field | Type | Description |
|-------|------|-------------|
| `keyset_id` | hex | |
| `B_` or `Y` | hex | |
| `status` | string | `INCLUDED`, `NOT_FOUND`, or `PENDING_EPOCH` |
| `proof` | `MerkleSumProof` \| null | Only for closed epochs |

### `MintCommitment` (optional)

| Field | Type | Description |
|-------|------|-------------|
| `keyset_id` | hex | |
| `B_` | hex | Blinded message |
| `amount` | uint | Token amount |
| `epoch_date` | string | Target epoch |
| `timestamp` | uint | Commitment time |
| `signature` | hex | Mint signature over fields |

### `PoLCommitmentResponse`

| Field | Type | Description |
|-------|------|-------------|
| `keyset_id` | hex | |
| `B_` | hex | |
| `status` | string | `PENDING_EPOCH` or `ALREADY_CLOSED` |
| `commitment` | `MintCommitment` \| null | Only for pending tokens |

## Epochs

- **Open epoch**: Current UTC day. Proofs return `PENDING_EPOCH` status (not queryable).
- **Closed epoch**: Past days. Finalized with fixed-size tree and OTS proof.

Closed epochs use **fixed padding** (configurable, default $2^{24}$ leaves) to hide user count.

### Epoch Retention

Epochs have a configurable retention period (default: 24 months). After expiration:

1. The `expires_at` timestamp is reached
2. Mint MAY prune the epoch report and associated proofs
3. Pruned epochs are no longer queryable

This aligns with keyset rotation: old keysets should be recycled before their epochs expire.

## Canonical Hashing

### Node Preimages

All preimages are UTF-8 strings with `:` (0x3A) separator.

| Node Type | Preimage Format |
|-----------|-----------------|
| Leaf | `<value>:<amount>` |
| Internal | `<left_hash>:<right_hash>:<sum>` |
| Empty | `EMPTY:0` → `df3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119` |

Where `<amount>` and `<sum>` are decimal strings (no leading zeros except `0`).

### Report Hash

1. Build JSON with keys in **alphabetical order**: `burn_root_amount`, `burn_root_hash`, `cumulative_burned`, `cumulative_minted`, `epoch_date`, `epoch_end`, `epoch_start`, `keyset_id`, `mint_root_amount`, `mint_root_hash`, `outstanding_balance`, `previous_epoch_hash`, `total_burned`, `total_minted`
2. Serialize compact (no whitespace)
3. `report_hash = SHA256(utf8_bytes)`

Rules: strings double-quoted, integers unquoted, `null` literal.

> **Note**: Implementers SHOULD construct the JSON string manually or use a canonical JSON library (e.g., JCS per [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785)) to ensure deterministic output.

### Report Signature

The mint MUST sign every `EpochReport` using Schnorr (BIP-340):

```
report_signature = schnorr_sign(mint_private_key, bytes.fromhex(report_hash))
```

Wallets SHOULD verify the signature before trusting the report:

```
verify_report_signature(report, mint_pubkey):
    hash_bytes = bytes.fromhex(report.report_hash)
    pubkey_xonly = extract_xonly(mint_pubkey)
    return schnorr_verify(pubkey_xonly, report.report_signature, hash_bytes)
```

This prevents a compromised server from serving altered historical data without detection.

## Merkle Sum Tree

### Construction

```
This implementation uses a Dense Merkle Tree with padding. To preserve privacy and prevent the padding from being predictably grouped, the leaves MUST be deterministically shuffled before the tree is built.

1. Sort actual transaction leaves by value (lexicographic)
2. Pad the list to 2^n with empty nodes
3. Deterministically shuffle all leaves using `previous_epoch_hash` as the RNG seed (use a constant like "genesis_seed" for the first epoch)
4. Build bottom-up: parent = combine(left, right)
```

### Proof Verification

```
verify(leaf_value, leaf_amount, siblings, root_hash, root_amount):
    h = SHA256(leaf_value + ":" + leaf_amount)
    a = leaf_amount
    for (sh, sa, pos) in siblings:
        a = a + sa
        h = SHA256(sh + ":" + h + ":" + a) if pos=="L" else SHA256(h + ":" + sh + ":" + a)
    return h == root_hash AND a == root_amount
```

### Privacy

Mints **MUST** pad closed epochs to fixed size ($2^{N}$ leaves) for k-anonymity. Tree size remains constant across all epochs to prevent users or observers from inferring the mint's transaction volume or traffic metadata. The deterministic shuffle ensures empty nodes and actual tokens are indistinguishable in the proof path.

## Epoch Chaining

```
Genesis: previous_epoch_hash = null
Epoch[n]: previous_epoch_hash = Epoch[n-1].report_hash
          cumulative_minted = Epoch[n-1].cumulative_minted + total_minted
          cumulative_burned = Epoch[n-1].cumulative_burned + total_burned
```

## OpenTimestamps

On epoch close, `report_hash` is submitted to OTS calendars. Wallets download `.ots` and verify independently.

## Endpoints

| Method | Path | Query | Response |
|--------|------|-------|----------|
| GET | `/v1/pol/roots/{keyset_id}` | `epoch_date?` | `PoLRootsResponse` |
| GET | `/v1/pol/history/{keyset_id}` | `limit?` (default 30) | `PoLHistoryResponse` |
| GET | `/v1/pol/verify/mint/{keyset_id}/{B_}` | `epoch_date?` | `PoLVerifyResponse` |
| GET | `/v1/pol/verify/burn/{keyset_id}/{Y}` | `epoch_date?` | `PoLVerifyResponse` |
| GET | `/v1/pol/commitment/{keyset_id}/{B_}` | `amount` | `PoLCommitmentResponse` |
| GET | `/v1/pol/ots/{keyset_id}/{epoch_date}` | — | Binary `.ots` file |

### OTS Download Headers

- `Content-Type: application/octet-stream`
- `X-Report-Hash`: timestamped hash
- `X-OTS-Confirmed`: `"true"` \| `"false"`

## Verification

### Token State

```
1. GET /v1/pol/verify/mint/{keyset_id}/{B_} → status?
   - NOT_FOUND: invalid token
   - PENDING_EPOCH: minted today, wait for epoch close
   - INCLUDED: continue
2. Y = hash_to_curve(secret)  [NUT-00]
3. GET /v1/pol/verify/burn/{keyset_id}/{Y} → status?
   - NOT_FOUND or PENDING_EPOCH: token is UNSPENT
   - INCLUDED: token is SPENT
```

### Mint Commitment (optional)

For tokens minted in the open epoch, the mint MAY return a signed commitment:

```
commitment = sign(keyset_id || B_ || amount || epoch_date || timestamp)
```

If the token is missing from tomorrow's closed tree, the wallet can present the commitment as proof of mint misbehavior.

### Chain Integrity

```
for i in 1..len(epochs):
    assert epochs[i].previous_epoch_hash == epochs[i-1].report_hash
    assert epochs[i].cumulative_minted == epochs[i-1].cumulative_minted + epochs[i].total_minted
    assert epochs[i].cumulative_burned == epochs[i-1].cumulative_burned + epochs[i].total_burned
```

## Mint Info

```json
{ "28": { "supported": true, "tree_size": 16777216, "retention_months": 24 } }
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tree_size` | uint | $2^{24}$ | Fixed padding size for k-anonymity |
| `retention_months` | uint | 24 | Months before epochs are pruned (0 = never) |

## Security

1. **Privacy**: Proof queries reveal token ownership to mint
2. **Trust**: PoL reduces but doesn't eliminate trust; OTS mitigates backdating
3. **OTS**: Calendar reliance; final verification against Bitcoin blockchain


## Issues and Limitations

While this PoL implementation provides strong cryptographic assurances, it has inherent limitations by design:

1. **Open Epoch Delay:** Tokens minted during the current UTC day (Open Epoch) cannot be cryptographically verified against a final root hash until the epoch closes. The optional `MintCommitment` mitigates this, but requires trusting the mint's signature temporarily.
2. **Mint as a Single Source of Truth:** Wallets must query the mint directly to obtain inclusion proofs. A malicious mint could theoretically attempt a *Split-View Attack* (serving different trees to different users). This is mitigated by OpenTimestamps anchoring, forcing the mint to commit to a single public canonical history.
3. **Privacy via IP Leaks:** Requesting an inclusion proof for a specific token reveals to the mint that the requesting IP is interested in that token. Wallets SHOULD route these requests through Tor, VPNs, or Nostr relays to preserve network-level privacy.
4. **Dense Tree Validation Overhead:** This protocol uses a Merkle Tree instead of a Sparse Tree one to remain database-independent and simplifies implementation. While this ensures that issuing servers do not need specialized key-value stores to compute the tree, generating the tree with millions of filled leaves requires sufficient memory allocation during the epoch-closing process.

[00]: 00.md
[06]: 06.md
[tests]: tests/x-tests.md
