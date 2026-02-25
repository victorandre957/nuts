# NUT-X Test Vectors

## Canonical Serialization

All values are serialized as UTF-8 strings. Fields are concatenated with the ASCII colon character (`:`, byte `0x3A`).

### Leaf Node

**Input**:
- `value` = `"02abc123"`
- `amount` = `8`

**Preimage** (UTF-8 string):
```text
02abc123:8
```

**Preimage** (hex bytes):
```
30 32 61 62 63 31 32 33 3a 38
```

**Hash**:
```
SHA256("02abc123:8") = <implementation computes this>
```

### Empty Node (Constant)

**Preimage**:
```
EMPTY:0
```

**Preimage** (hex bytes):
```
45 4d 50 54 59 3a 30
```

**Hash**:
```
SHA256("EMPTY:0") = df3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119
```

### Internal Node

**Input**:
- `left.hash` = `"abc123..."` (64 hex chars)
- `left.amount` = `8`
- `right.hash` = `"def456..."` (64 hex chars)
- `right.amount` = `4`

**Preimage** (template):
```
<left_hash>:<right_hash>:<total_amount>
```

**Example**:
```
abc123...:def456...:12
```

Where `:` is ASCII 0x3A and `12` is the decimal string of `8 + 4`.

## Merkle Sum Tree

### Tree Construction Example

For privacy, mints SHOULD pad trees to a fixed size (recommended $2^{24}$).

Example with 3 actual leaves padded to size 4, using a hypothetical epoch hash "seed123":

**Actual leaves:**
```json
[
  ["B_1", 100],
  ["B_2", 50],
  ["B_3", 25]
]
```

Deterministically Shuffled list (hypothetical result for "seed123"):

```
[
  ["B_2", 50],
  ["EMPTY", 0],
  ["B_1", 100],
  ["B_3", 25]
]
```

**Padded tree (size 4):**
```
                    Root (amount=175)
                   /                \
           Node1 (50)             Node2 (125)
          /        \             /         \
       B_2(50)   EMPTY(0)     B_1(100)    B_3(25)
```

The empty node uses:
- `hash = SHA256("EMPTY:0")` = `df3f619804a92fdb4057192dc43dd748ea778adc52bc498ce80524c014b81119`
- `amount = 0`

**Result:** The root amount (175) correctly reflects the sum of actual tokens, while the tree depth remains constant regardless of the number of tokens.

## Merkle Sum Proof Verification

### Example Tree

Build a tree with 4 leaves:

```json
[
  ["B_1", 2],
  ["B_2", 4],
  ["B_3", 8],
  ["B_4", 1]
]
```

The tree structure:

```
                    Root (amount=15)
                   /                \
           Node1 (6)              Node2 (9)
          /        \             /        \
       B_1(2)    B_2(4)      B_3(8)     B_4(1)
```

### Valid Inclusion Proof for B_3

```json
{
  "leaf_value": "B_3",
  "leaf_amount": 8,
  "siblings": [
    ["<hash_of_B_4>", 1, "R"],
    ["<hash_of_Node1>", 6, "L"]
  ],
  "root_hash": "<root_hash>",
  "root_amount": 15
}
```

Verification steps:
1. `current = LeafNode("B_3", 8)`
2. Combine with B_4 (right sibling): `current = combine(current, B_4_node)`
3. Combine with Node1 (left sibling): `current = combine(Node1, current)`
4. Check: `current.hash == root_hash AND current.amount == 15`

## Epoch Report Hash

### Canonical JSON Serialization

The `report_hash` uses compact JSON with:
- Keys in **alphabetical order**
- No whitespace between tokens
- Strings: double-quoted
- Integers: decimal, no quotes
- Null: literal `null`

### Example

**Epoch Report Data**:
```json
{
  "burn_root_amount": 12500,
  "burn_root_hash": "789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456",
  "cumulative_burned": 112500,
  "cumulative_minted": 150000,
  "epoch_date": "2024-01-15",
  "epoch_end": 1705363199,
  "epoch_start": 1705276800,
  "keyset_id": "009a1f293253e41e",
  "mint_root_amount": 50000,
  "mint_root_hash": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5",
  "outstanding_balance": 37500,
  "previous_epoch_hash": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "total_burned": 12500,
  "total_minted": 50000
}
```

**Serialized (no whitespace)**:
```
{"burn_root_amount":12500,"burn_root_hash":"789abcdef...","cumulative_burned":112500,...}
```

**Hash**:
```
report_hash = SHA256(serialized_json_utf8_bytes)
```

### Genesis Epoch (null previous_epoch_hash)

When `previous_epoch_hash` is null:
```json
{
  ...
  "previous_epoch_hash": null,
  ...
}
```

Serialized as literal `null` (no quotes):
```
..."previous_epoch_hash":null,...
```

## Chain Validation

### Valid Chain

```json
[
  {
    "epoch_date": "2024-01-13",
    "previous_epoch_hash": null,
    "cumulative_minted": 75000,
    "cumulative_burned": 62500,
    "total_minted": 75000,
    "total_burned": 62500,
    "report_hash": "epoch1_hash"
  },
  {
    "epoch_date": "2024-01-14",
    "previous_epoch_hash": "epoch1_hash",
    "cumulative_minted": 100000,
    "cumulative_burned": 100000,
    "total_minted": 25000,
    "total_burned": 37500,
    "report_hash": "epoch2_hash"
  },
  {
    "epoch_date": "2024-01-15",
    "previous_epoch_hash": "epoch2_hash",
    "cumulative_minted": 150000,
    "cumulative_burned": 112500,
    "total_minted": 50000,
    "total_burned": 12500,
    "report_hash": "epoch3_hash"
  }
]
```

Validation:
1. Epoch 1: `previous_epoch_hash == null` ✓ (genesis)
2. Epoch 2: `previous_epoch_hash == epoch1_hash` ✓
3. Epoch 3: `previous_epoch_hash == epoch2_hash` ✓
4. Cumulative values: 
   - Epoch 2: `100000 == 75000 + 25000` ✓, `100000 == 62500 + 37500` ✓
   - Epoch 3: `150000 == 100000 + 50000` ✓, `112500 == 100000 + 12500` ✓

Chain is **valid**.

### Invalid Chain (Hash Mismatch)

```json
[
  {
    "epoch_date": "2024-01-13",
    "previous_epoch_hash": null,
    "report_hash": "epoch1_hash"
  },
  {
    "epoch_date": "2024-01-14",
    "previous_epoch_hash": "wrong_hash",
    "report_hash": "epoch2_hash"
  }
]
```

Validation fails: `epoch2.previous_epoch_hash != epoch1.report_hash`

### Invalid Chain (Cumulative Mismatch)

```json
[
  {
    "epoch_date": "2024-01-13",
    "cumulative_minted": 75000,
    "total_minted": 75000,
    "report_hash": "epoch1_hash"
  },
  {
    "epoch_date": "2024-01-14",
    "previous_epoch_hash": "epoch1_hash",
    "cumulative_minted": 90000,
    "total_minted": 25000
  }
]
```

Validation fails: `90000 != 75000 + 25000`

## API Response Examples

### GET /v1/pol/roots/{keyset_id}

Current epoch (not closed):

```json
{
  "keyset_id": "009a1f293253e41e",
  "epoch_date": "2024-01-15",
  "is_closed": false,
  "previous_epoch_hash": "a1b2c3...",
  "mint_root_hash": "d4e5f6...",
  "mint_root_amount": 50000,
  "burn_root_hash": "789abc...",
  "burn_root_amount": 12500,
  "outstanding_balance": 37500,
  "cumulative_minted": 150000,
  "cumulative_burned": 112500,
  "report_timestamp": 1705312800,
  "report_hash": "def012...",
  "report_signature": "abc987...",
  "ots_proof": null,
  "ots_confirmed": false
}
```

### GET /v1/pol/verify/mint/{keyset_id}/{B_}

Token found:

```json
{
  "keyset_id": "009a1f293253e41e",
  "B_": "02abc123def456...",
  "status": "INCLUDED",
  "proof": {
    "leaf_value": "02abc123def456...",
    "leaf_amount": 8,
    "siblings": [
      ["feedface...", 4, "R"],
      ["deadbeef...", 12, "L"]
    ],
    "root_hash": "d4e5f6...",
    "root_amount": 50000
  }
}
```

Token minted today (Open Epoch):

```json
{
  "keyset_id": "009a1f293253e41e",
  "B_": "02abc123def456...",
  "status": "PENDING_EPOCH",
  "proof": null
}
```

Token not found:

```json
{
  "keyset_id": "009a1f293253e41e",
  "B_": "02xyz789...",
  "status": "NOT_FOUND",
  "proof": null
}
```

## Token State Verification

### Determine Token Status

Given a token with:
- `B_` = `"02abc123..."`
- `secret` = `"mysecret123"`
- `Y` = `hash_to_curve("mysecret123")` = `"03def456..."`

Query sequence:

1. `GET /v1/pol/verify/mint/{keyset_id}/02abc123...`
   - Response: `{"status": "INCLUDED", ...}`
   - Token was minted ✓

2. `GET /v1/pol/verify/burn/{keyset_id}/03def456...`
   - If `{"status": "PENDING_EPOCH"}` → Token is **UNSPENT** (valid, can be redeemed)
   - If `{"status": "INCLUDED"}` → Token is **SPENT** (already redeemed)

## Report Signature Verification

### Signature Format

The `report_signature` is a BIP-340 Schnorr signature over the `report_hash`:

```
report_signature = schnorr_sign(mint_private_key, bytes.fromhex(report_hash))
```

### Verification Steps

1. Extract mint's x-only public key from the compressed public key:
   ```
   mint_pubkey = "0296d0aa13b6a31cf0cd974249f28c7b7176d7274712c95a41c7d8066d3f29d679"
   xonly_pubkey = mint_pubkey[2:]  # Remove 02/03 prefix
   ```

2. Convert report_hash and signature from hex to bytes:
   ```
   hash_bytes = bytes.fromhex(report.report_hash)
   sig_bytes = bytes.fromhex(report.report_signature)
   ```

3. Verify using BIP-340 Schnorr:
   ```
   is_valid = schnorr_verify(xonly_pubkey, sig_bytes, hash_bytes)
   ```

### Valid Signature Example

Given:
- `report_hash` = `"a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"`
- `mint_pubkey` = `"0296d0aa13b6a31cf0cd974249f28c7b7176d7274712c95a41c7d8066d3f29d679"`
- `report_signature` = valid Schnorr signature

Verification:
```python
from coincurve import PublicKey, PublicKeyXOnly

pubkey = PublicKey(bytes.fromhex(mint_pubkey))
xonly = PublicKeyXOnly(pubkey.format()[1:])
is_valid = xonly.verify(bytes.fromhex(report_signature), bytes.fromhex(report_hash))
# is_valid == True
```

### Invalid Signature Cases

1. **Wrong signature**: signature created with different key → `is_valid == False`
2. **Tampered hash**: report_hash modified after signing → `is_valid == False`
3. **Empty signature**: `report_signature == ""` → verification should return `False`

### Wallet Verification Requirement

Wallets SHOULD verify the signature before trusting any epoch report data. This prevents:
- Compromised servers serving altered historical data
- Man-in-the-middle attacks on report delivery
- Replay attacks with modified report contents

## Epoch Expiration

### expires_at Calculation

Epochs have an expiration timestamp calculated from `epoch_date + retention_months`:

```python
def calculate_expires_at(epoch_date: str, retention_months: int) -> int | None:
    """
    Calculate expiration timestamp for an epoch.
    
    Args:
        epoch_date: "YYYY-MM-DD" format
        retention_months: Months to retain (0 = never expires)
    
    Returns:
        Unix timestamp when epoch expires, or None if disabled
    """
    if retention_months <= 0:
        return None
    
    dt = datetime.strptime(epoch_date, "%Y-%m-%d")
    year = dt.year + (dt.month + retention_months - 1) // 12
    month = (dt.month + retention_months - 1) % 12 + 1
    max_day = calendar.monthrange(year, month)[1]
    day = min(dt.day, max_day)
    expires_dt = datetime(year, month, day, 23, 59, 59, tzinfo=timezone.utc)
    return int(expires_dt.timestamp())
```

### Test Vectors

| epoch_date | retention_months | expires_at (approx) |
|------------|------------------|---------------------|
| 2024-01-15 | 24 | 2026-01-15 23:59:59 UTC |
| 2024-01-31 | 1 | 2024-02-29 23:59:59 UTC (leap year) |
| 2024-10-15 | 6 | 2025-04-15 23:59:59 UTC |
| 2024-01-15 | 0 | null (never expires) |

### is_expired Check

```python
def is_expired(report: EpochReport) -> bool:
    if report.expires_at is None:
        return False
    return int(datetime.now(timezone.utc).timestamp()) > report.expires_at
```

## Configurable Tree Size

### Tree Size from Settings

Mints can configure the fixed tree size via `pol_tree_size_exponent`:

```python
tree_size = 2 ** settings.pol_tree_size_exponent
# Default: 2^24 = 16,777,216 leaves
```

### Test Vectors

| pol_tree_size_exponent | tree_size | Memory (approx) |
|------------------------|-----------|-----------------|
| 20 | 1,048,576 | ~100 MB |
| 22 | 4,194,304 | ~400 MB |
| 24 | 16,777,216 | ~1.6 GB |
| 26 | 67,108,864 | ~6 GB |

### Custom Tree Size in MerkleSumTree

```python
# Using custom fixed_tree_size (overrides default)
tree = MerkleSumTree(
    leaves=[("B_1", 100), ("B_2", 200)],
    epoch_date="2024-01-15",
    use_fixed_padding=True,
    fixed_tree_size=2**20
)
assert tree.padded_size == 1048576
```
