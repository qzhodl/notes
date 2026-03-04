# PyQuarkChain DB Storage Layout Notes

## Overview

There are **separate DB instances** for each chain component:

- **Root DB** — one shared instance, managed by the master process
- **Shard DB** — one instance per shard, managed by each slave process

Keys are constructed by prefixing a tag with a hash or number.

---

## Root Chain DB (`RootDb`)

**File:** `quarkchain/cluster/root_state.py` — `RootDb` class

| DB Key | Value | Written By | Purpose |
|---|---|---|---|
| `b"tipHash"` | 32-byte block hash | `update_tip_hash()` | Current best chain tip |
| `b"rblock_" + hash` | serialized `RootBlock` | `put_root_block()` | Full root block |
| `b"lastlist_" + hash` | serialized `LastMinorBlockHeaderList` | `put_root_block()` | Last minor header per shard at this root block |
| `b"ri_%d" % height` | 32-byte block hash | `put_root_block_index()` | Height → hash (best chain index) |
| `b"m_r_" + minor_hash + shard_id_bytes` | 32-byte root hash | `put_root_block_index()` | Which root block confirmed this minor block |
| `b"count_%d_%d" % (shard_id, height)` | packed miner→count bytes | `put_root_block_index()` | Miner block count per shard (for PoSW) |
| `b"mheader_" + minor_hash` | serialized `TokenBalanceMap` | `put_minor_block_coinbase()` | Validated stamp + coinbase tokens of a minor block |

---

## Shard Chain DB (`ShardDbOperator`)

**File:** `quarkchain/cluster/shard_db_operator.py` — `ShardDbOperator` class

| DB Key | Value | Written By | Purpose |
|---|---|---|---|
| `b"rblock_" + hash` | serialized `RootBlock` | `put_root_block()` | Local copy of root blocks (shard needs them for x-shard tx) |
| `b"r_last_m" + root_hash` | minor header hash | `put_root_block()` | Latest minor block confirmed by this root block in this shard |
| `b"genesis_" + root_hash` | serialized `MinorBlock` | `put_genesis_block()` | Genesis minor block for this shard |
| `b"mblock_" + hash` | serialized `MinorBlock` | `put_minor_block()` | Full minor block |
| `b"mi_%d" % height` | 32-byte minor block hash | `put_minor_block_index()` | Height → hash (best shard chain index) |
| `b"tx_count_" + mblock_hash` | 4-byte cumulative int | `put_total_tx_count()` | Running total tx count up to this block |
| `b"commit_" + hash` | empty | `put_minor_block_confirmed()` | Minor block finalized by root chain |
| `b"txindex_" + tx_hash` | mblock_hash + tx_index | `put_transaction_index()` | TX hash → which block/index it is in |
| `b"xShard_" + mblock_hash` | serialized `CrossShardTransactionList` | `put_minor_block_xshard_tx_list()` | Outgoing x-shard deposits from this minor block |
| `b"xd_" + mblock_hash` | serialized `HashList` | `__put_xshard_deposit_hash_list()` | Hashes of incoming x-shard deposits |
| `b"xr_" + mblock_hash` | serialized `CrossShardTransactionList` | `put_confirmed_cross_shard_transaction_deposit_list()` | Confirmed incoming x-shard deposits |
| `b"index_addr_" + recipient + height + flag + idx` | empty | tx history index | Per-address transaction lookup index |
| `b"index_alltx_" + height + flag + idx` | empty | tx history index | Global transaction lookup index |
| `b"rb_committing"` | 32-byte hash | `write_committing_hash()` | Crash-recovery: root block being committed |

> **Note:** Both Root DB and Shard DB use `b"rblock_" + hash`. They do **not** conflict because each shard has an entirely separate RocksDB instance on disk.

---

## Q&A

### Q: Is there a single place to look up all DB key patterns?

No centralized key registry exists. The two authoritative files are:

- `quarkchain/cluster/root_state.py` — `RootDb` class (all root chain keys)
- `quarkchain/cluster/shard_db_operator.py` — `ShardDbOperator` class (all shard chain keys)

Search for `self.db.put(b"` inside those two files to find every key definition.

---

### Q: What is `LastMinorBlockHeaderList`?

**It is NOT all headers in a root block.** It is the **last (highest-height) `MinorBlockHeader` per shard** included in a given root block.

**Example:**
```
RootBlock N includes:
  shard 0: heights 10, 11, 12
  shard 1: heights 5, 6

lastlist_<hash of N>:
  shard 0 → header at height 12   ← the tail
  shard 1 → header at height 6    ← the tail
```

**Why it is needed:**
When validating the *next* root block (N+1), the code checks that each shard's first block in N+1 directly continues from the tail stored in N:

```python
# root_state.py validate_block()
prev_last = db.get_root_block_last_minor_block_header_list(prev_root_hash)
# prepend the tail, then verify the chain is gapless
headers = [prev_tail] + new_headers
for i in range(len(headers) - 1):
    assert headers[i+1].hash_prev_minor_block == headers[i].get_hash()
```

**The invariant:** shard block chains must be **gapless** across root blocks. No shard block can be skipped.

```
Root N     → lastlist = {shard0: h12, shard1: h6}
                               ↓              ↓
Root N+1   must start with    h13            h7
           h13.hash_prev_minor_block == h12.get_hash()  ✓
```

---

### Q: Why does the root chain care about shard coinbase amounts?

**Coinbase** = the block reward — newly minted tokens paid to a miner when they produce a valid block. This is how new coins enter circulation (not a transfer of existing tokens).

QuarkChain has two types:

| | Shard miner | Root miner |
|---|---|---|
| Mines | Minor block | Root block |
| Earns | Shard coinbase (`coinbase_amount_map` in `MinorBlockHeader`) | Root coinbase |

**The root miner's reward is calculated FROM the shard coinbase amounts:**

```python
# root_state.py _calculate_root_block_coinbase()
reward_tax_rate = config.reward_tax_rate   # e.g. 0.5

# Sum coinbase of all minor blocks confirmed by this root block
for m_hash in m_hash_list:
    total += db.get_minor_block_coinbase_tokens(m_hash)  # reads b"mheader_" + hash

# Root miner earns a share proportional to reward_tax_rate
# + a fixed base amount (coinbase_amount * decay_factor^epoch)
```

Example at `reward_tax_rate = 0.5`: root miner earns the same amount as all shard miners combined, plus a fixed base.

**This is why `put_minor_block_coinbase()` exists:** the root master does not store full minor blocks (too expensive). It only stores the coinbase token amounts — just enough to calculate the root block reward.

---

### Q: What does `put_minor_block_coinbase()` actually do?

It serves **two purposes with one DB write** at key `b"mheader_" + minor_hash`:

1. **Validation stamp** — presence of this key means "a shard slave validated this block." `contain_minor_block_by_hash()` checks for it; a minor block cannot appear in a root block without the stamp.

2. **Stores coinbase token amounts** — the value is a serialized `TokenBalanceMap` used later by `_calculate_root_block_coinbase()`.

**The flow:**
```
Shard slave validates minor block
    ↓
Slave sends AddMinorBlockHeaderRequest to master (RPC)
    ↓
master.py handle_add_minor_block_header_request()
    ↓
root_state.add_validated_minor_block_hash(hash, tokens)
    ↓
db.put(b"mheader_" + hash, TokenBalanceMap(tokens).serialize())
    ↓
Root miner can now include this block in a root block ✓
Root miner's reward includes a share of these tokens ✓
```