# Erigon MDBX Flat Storage: Visual Explanation

---

## Traditional Trie-Based Storage (Geth, Nethermind, Besu)

### How Data is Stored on Disk

```
┌─────────────────────────────────────────────────────────────┐
│                    BLOCKCHAIN STATE                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Merkle Patricia Trie (MPT)                      │
│                                                              │
│         Root Hash: 0xabcd1234...                            │
│              /           \                                   │
│          Branch         Branch                              │
│         /     \         /     \                             │
│     Leaf    Leaf    Leaf    Leaf                           │
│   (Account) (Account) (Account) (Account)                  │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  DATABASE (LevelDB/RocksDB)                  │
│                                                              │
│  Key: 0x1a2b...  →  Value: [Branch Node Data]              │
│  Key: 0x3c4d...  →  Value: [Leaf: Account A]               │
│  Key: 0x5e6f...  →  Value: [Leaf: Account B]               │
│  Key: 0x7g8h...  →  Value: [Branch Node Data]              │
│  Key: 0x9i0j...  →  Value: [Leaf: Account C]               │
│                                                              │
│  STORED: Every trie node as separate key-value pair         │
│  Problem: MASSIVE redundancy—trie nodes repeated            │
│           across blocks for slightly different states       │
└─────────────────────────────────────────────────────────────┘

💾 Disk Usage: 12+ TB for archive nodes
```

### Data Flow: Reading Account Balance

```
Query: "What's the balance of 0xAlice at block 18,000,000?"

1. Load State Root Hash for block 18,000,000
2. Traverse Trie: Root → Branch → Branch → Leaf
   ├─ Read node: 0x1a2b... (4 KB)
   ├─ Read node: 0x3c4d... (4 KB)
   ├─ Read node: 0x5e6f... (4 KB)
   └─ Read node: 0x7g8h... (4 KB)  ← Account found!

Total: 4 disk reads, ~16 KB read
⏱️  Latency: Slower due to multiple random disk accesses
```

---

## Erigon Flat Storage (MDBX Architecture)

### How Data is Stored on Disk

```
┌─────────────────────────────────────────────────────────────┐
│                    BLOCKCHAIN STATE                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    NO TRIE ON DISK! ❌                       │
│           (Tries reconstructed in-memory only)               │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              FLAT STORAGE (MDBX Database)                    │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │  ACCOUNTS TABLE (Temporal/Historical)             │      │
│  ├──────────────────────────────────────────────────┤      │
│  │ Address       │ Block Range  │ Balance │ Nonce   │      │
│  ├──────────────────────────────────────────────────┤      │
│  │ 0xAlice...    │ 0-1,000,000  │ 10 ETH  │ 5       │      │
│  │ 0xAlice...    │ 1,000,001-∞  │ 15 ETH  │ 6  ← Latest    │
│  │ 0xBob...      │ 0-500,000    │ 5 ETH   │ 1       │      │
│  │ 0xBob...      │ 500,001-∞    │ 3 ETH   │ 2  ← Latest    │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │  STORAGE TABLE (Smart Contract Storage)           │      │
│  ├──────────────────────────────────────────────────┤      │
│  │ Contract   │ Slot  │ Block Range │ Value         │      │
│  ├──────────────────────────────────────────────────┤      │
│  │ 0xUniswap  │ 0x00  │ 0-2M        │ 0x1234        │      │
│  │ 0xUniswap  │ 0x00  │ 2M-∞        │ 0x5678 ← Latest      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
│  STORED: Only account data + change history                 │
│  Benefit: No redundant trie nodes!                          │
└─────────────────────────────────────────────────────────────┘

💾 Disk Usage: 2.5-6 TB for archive nodes (50-70% less!)
```

### Data Flow: Reading Account Balance

```
Query: "What's the balance of 0xAlice at block 18,000,000?"

1. Direct lookup in ACCOUNTS table:
   ├─ Find 0xAlice WHERE block_range contains 18,000,000
   └─ Return balance: 15 ETH

Total: 1 disk read, ~256 bytes read
⚡ Latency: MUCH faster—single direct lookup

Memory-mapped file = data accessed via mmap(), OS handles caching
```

### When Trie is Needed (e.g., for block validation)

```
On-The-Fly Trie Reconstruction:

┌──────────────────────────────────┐
│  Need State Root Hash?           │
│  (e.g., validating a block)      │
└──────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────┐
│  1. Read relevant accounts       │
│     from flat storage            │
└──────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────┐
│  2. Build Merkle Patricia Trie   │
│     IN MEMORY (RAM)              │
└──────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────┐
│  3. Calculate Root Hash          │
│     0xabcd1234...                │
└──────────────────────────────────┘
            │
            ▼
        ✅ Verified

Note: Trie exists temporarily in RAM, never written to disk!
```

---

## Side-by-Side Comparison

### Storing Account State Change (e.g., Alice sends 1 ETH)

**Traditional Trie-Based:**
```
OLD STATE (Block N-1):
  Trie Node A → [stored on disk]
  Trie Node B → [stored on disk]
  Trie Node C (Alice: 15 ETH) → [stored on disk]

NEW STATE (Block N):
  Trie Node A' → [NEW, stored on disk]  ← Changed!
  Trie Node B' → [NEW, stored on disk]  ← Changed!
  Trie Node C' (Alice: 14 ETH) → [NEW, stored on disk]  ← Changed!

Result: Must store 3+ new trie nodes for ONE account change
Storage Cost: ~12-16 KB per account update
```

**Erigon Flat Storage:**
```
OLD STATE:
  0xAlice | blocks 0-N-1 | 15 ETH | nonce 6

NEW STATE:
  0xAlice | blocks N-∞   | 14 ETH | nonce 7

Result: Add one new row with new balance + block range
Storage Cost: ~256 bytes per account update
Savings: 98% less disk write for same state change! 🚀
```

---

## Memory-Mapped Database (MDBX) Magic

### What is Memory Mapping?

```
Traditional Database I/O:
┌──────────┐    read()     ┌──────────┐    memcpy    ┌──────────┐
│   Disk   │  ─────────►   │  Kernel  │  ─────────►  │   App    │
│  Storage │               │  Buffer  │              │  Memory  │
└──────────┘               └──────────┘              └──────────┘
  (slow)                      (copy)                  (your RAM)

Problem: Double buffering, extra memory copy


MDBX Memory-Mapped I/O:
┌──────────┐               ┌──────────┐
│   Disk   │   mmap() ───► │   App    │
│  Storage │   (direct)    │  Memory  │
└──────────┘               └──────────┘
  (slow)                      (direct access!)

Magic: OS maps disk file directly into memory address space
       App reads data as if it's in RAM (zero-copy)
       OS handles paging/caching automatically
```

### Copy-on-Write (ACID Transactions)

```
Transaction Start:
┌─────────────────────────────────┐
│  Original Data Page (on disk)   │
│  [Alice: 15 ETH] [Bob: 5 ETH]   │
└─────────────────────────────────┘
          │
          │ Write happens
          ▼
┌─────────────────────────────────┐
│  NEW Page Created (COW)         │
│  [Alice: 14 ETH] [Bob: 5 ETH]   │  ← Changes written here
└─────────────────────────────────┘
          │
          │ Commit = make new page visible
          ▼
┌─────────────────────────────────┐
│  Original Page (old version)    │  ← Can be discarded or kept
│  [Alice: 15 ETH] [Bob: 5 ETH]   │     for historical queries
└─────────────────────────────────┘

Benefit: Atomic commits, no corruption, instant rollback
```

---

## Why Erigon is So Efficient: The Numbers

### Archive Node Storage Comparison

```
Storing 18 million blocks of Ethereum history:

Traditional Trie-Based (Geth):
├─ Trie nodes:        10 TB
├─ Block data:        2 TB
├─ Transaction data:  1 TB
└─ Total:            ~13 TB
   └─ Problem: Trie nodes dominate storage!

Erigon Flat Storage:
├─ Account history:   1.5 TB  ← Compressed efficiently
├─ Storage history:   1.0 TB  ← Temporal tables
├─ Block data:        2 TB    ← Same as Geth
├─ Transaction data:  1 TB    ← Same as Geth
└─ Total:            ~5.5 TB
   └─ Savings: 58% reduction! 🎉
```

### Why the Massive Difference?

```
Every block (~12 seconds) updates ~100-1000 accounts

Traditional Trie:
  For each changed account:
    - Store modified leaf node (~4 KB)
    - Store ALL parent branch nodes (4-6 nodes × 4 KB = 16-24 KB)
  
  Total per block: 100 accounts × 20 KB = 2 MB just for trie nodes!
  18 million blocks × 2 MB = 36 TB of trie data 😱
  (Pruning helps but archive still huge)

Erigon Flat:
  For each changed account:
    - Store account record with (address, block, balance, nonce)
    - Size: ~256 bytes
  
  Total per block: 100 accounts × 256 bytes = 25 KB
  18 million blocks × 25 KB = 450 GB
  
  Savings: 98.75% less redundant data! ✨
```

---

## RAM Requirements Trade-off

### Why Erigon Needs More RAM

```
Traditional Database:
┌────────────────────────────┐
│  Database manages memory   │
│  internally with cache     │
│  Typical: 4-8 GB cache     │
└────────────────────────────┘

MDBX Memory-Mapped:
┌────────────────────────────┐
│  OS manages memory pages   │
│  Entire DB can be "mapped" │
│  More RAM = more pages     │
│  cached = faster access    │
│                            │
│  Recommended: 16-32 GB RAM │
└────────────────────────────┘

Trade-off: Use more RAM → Save tons of disk space
          Perfect for modern servers!
```

---

## Real-World Performance

### Query Speed Comparison (Archive Node)

```
Query: "Get balance of 0xVitalik at block 15,000,000"

Geth (Traditional Trie):
├─ Seek to block 15M state root
├─ Read trie nodes: 4-6 disk seeks
├─ Decompress and parse nodes
└─ Time: ~50-200ms (depends on cache)

Erigon (Flat Storage):
├─ Direct table lookup with index
├─ Single disk seek (or RAM if cached)
└─ Time: ~1-10ms (10-20× faster! ⚡)


Query: "Get all transactions for 0xAddress"

Geth:
├─ Must scan blocks or use external index
└─ Time: Seconds to minutes

Erigon:
├─ Indexed temporal history
└─ Time: ~10-100ms (built-in fast queries!)
```

---

## Simplified Mental Model

### Think of it like...

**Traditional Trie Storage = Git with Full Snapshots**
```
Every commit (block) stores entire folder structure
  commit1/  → full tree (100 MB)
  commit2/  → full tree (100 MB)  ← 99% same as commit1!
  commit3/  → full tree (100 MB)  ← 99% same as commit2!
  
Result: Massive redundancy, huge disk usage
```

**Erigon Flat Storage = Git with Diffs**
```
Base state + series of diffs
  base/     → initial state (100 MB)
  diff1     → +changed file (1 MB)
  diff2     → +changed file (1 MB)
  
Result: Minimal redundancy, efficient storage
```

---

## Key Takeaways

✅ **Erigon eliminates trie node redundancy** by storing only account data  
✅ **Temporal tables** track historical changes efficiently  
✅ **Memory-mapping** provides zero-copy performance  
✅ **50-70% disk savings** on archive nodes (2.5-6 TB vs 12+ TB)  
✅ **Faster historical queries** due to direct flat table access  
✅ **Tries reconstructed on-demand** in RAM when needed for validation  
✅ **Trade-off: Higher RAM usage** but saves massive disk space  

**Perfect for:**
- Archive nodes (huge savings!)
- Historical data analysis
- Block explorers
- Research and analytics
- Anyone with limited disk but good RAM

**Not optimal for:**
- Severely RAM-constrained systems (<16 GB)
- Use cases needing only recent state (traditional pruned is fine)
