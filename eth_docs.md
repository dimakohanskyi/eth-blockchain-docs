# Ethereum Node Operations Guide

---

## Table of Contents
1. [Ethereum Basics](#ethereum-basics)
2. [How Ethereum Works Today](#how-ethereum-works-today)
3. [Node Types & Roles](#node-types--roles)
4. [Software Stack](#software-stack)
5. [Database Architectures](#database-architectures)
6. [Hardware & Network Requirements](#hardware--network-requirements)
7. [Validator Operations](#validator-operations)
8. [Economics & Rewards](#economics--rewards)
9. [Common Questions](#common-questions)
10. [Quick Reference](#quick-reference)

---

## Ethereum Basics

### What is Ethereum?
Ethereum is a **decentralized computer network** that runs programs (smart contracts) and keeps track of digital assets. Think of it as:
- A **shared ledger** that records all transactions
- A **global computer** that executes programs without a central authority
- A **currency system** with ETH as the native token

### Key Concepts

**Blockchain**  
A chain of "blocks" where each block contains:
- A batch of transactions (payments, smart contract calls)
- A reference to the previous block (forming the chain)
- A timestamp and other metadata

Think of it like pages in a ledger that are permanently linked together.

**Block**  
A new block is added approximately every **12 seconds**. Each block can contain:
- Simple ETH transfers
- Smart contract interactions
- Data and state changes

**Transactions**  
Any action on Ethereum is a transaction:
- Sending ETH from one account to another
- Calling a smart contract function
- Deploying new smart contracts

**Gas**  
Gas is the unit that measures computational work. Users pay gas fees (in ETH) to:
- Compensate validators for processing transactions
- Prevent spam and infinite loops
- Prioritize transactions (higher gas price = faster inclusion)

**Smart Contracts**  
Programs that live on the blockchain:
- Written in languages like Solidity
- Execute automatically when conditions are met
- Cannot be changed once deployed (immutable)
- Examples: DeFi protocols, NFTs, DAOs

**Accounts**  
Two types:
1. **Externally Owned Accounts (EOAs)**: Controlled by private keys (your wallet)
2. **Contract Accounts**: Controlled by smart contract code

---

## How Ethereum Works Today

### Proof of Stake (PoS)
*Ethereum switched from Proof of Work to Proof of Stake in September 2022 (The Merge)*

**Simple Explanation:**
Instead of miners competing with energy-intensive computations, validators with staked ETH take turns proposing blocks and voting on them.

**Key Differences:**

| Proof of Work (Old) | Proof of Stake (Current) |
|---------------------|--------------------------|
| Miners solve puzzles using computing power | Validators stake 32 ETH as collateral |
| First to solve wins the block reward | Randomly selected based on stake |
| High energy consumption | ~99.95% less energy |
| Hardware: GPUs/ASICs | Hardware: Regular servers |

### The Two-Client Architecture

Ethereum now requires **two pieces of software** working together:

**Execution Client (EL)**
- Runs the Ethereum Virtual Machine (EVM)
- Processes transactions and smart contracts
- Maintains account balances and contract storage
- Manages the mempool (pending transactions)
- Provides JSON-RPC API for applications

**Consensus Client (CL)**
- Handles Proof of Stake logic
- Manages validator committees
- Coordinates block proposal and attestation
- Determines chain finality
- Implements fork choice rules

**Why Both?**  
This separation allows each layer to be optimized independently while maintaining security and decentralization.

### How Blocks Are Created

**1. Slot System**
- Time is divided into **12-second slots**
- Each slot can contain one block
- One validator is randomly selected as the proposer for each slot

**2. Proposing (Building a Block)**

*Analogy: Like a presenter showing a slide to the class*

The selected validator:
1. Collects pending transactions from the mempool
2. Executes them to build a new block
3. Broadcasts the block to the network
4. Earns rewards for successful proposal

**3. Attesting (Voting on Blocks)**

*Analogy: Like classmates giving thumbs-up to confirm the slide is correct*

Simultaneously, other validators:
1. Verify the proposed block is valid
2. Vote (attest) that this block extends the correct chain
3. Sign their vote with their validator key
4. Broadcast attestations to the network

**What an Attestation Contains:**
- **Head vote**: "This block is the current best chain tip"
- **Source checkpoint**: The last justified (partially confirmed) point
- **Target checkpoint**: The new checkpoint we're confirming
- **Signature**: Cryptographic proof from the validator

**4. Finality**

After enough attestations (about ⅔ of total staked ETH) agree on checkpoints:
- Those checkpoints become **final**
- They cannot be reversed (provides strong security)
- Typically takes 2-3 epochs (~12-19 minutes)

### Epochs
- An **epoch** = 32 slots = ~6.4 minutes
- Used for validator committee assignments
- Used for calculating checkpoint finality
- Used for reward/penalty calculations

---

## Node Types & Roles

### 1. Full Node (Non-Validating)

**What it does:**
- Runs both Execution Client (EL) and Consensus Client (CL)
- Downloads and verifies every block
- Stores recent blockchain state
- Serves data to peers and applications
- Does NOT propose or attest to blocks

**Who needs it:**
- DApp developers needing reliable RPC access
- Exchanges and custodians
- Block explorers and analytics services
- Anyone wanting to verify the chain independently

**Requirements:**
- No ETH stake required
- Moderate hardware (see hardware section)
- Good internet connection

### 2. Validator Node

**What it does:**
- Everything a full node does, PLUS:
- Proposes blocks when selected
- Attests to other validators' blocks
- Participates in consensus
- Earns staking rewards

**Who needs it:**
- Solo stakers with 32 ETH
- Staking-as-a-service operators
- Anyone wanting to secure the network and earn rewards

**Requirements:**
- **32 ETH** locked as stake (or participation in pool/DVT)
- Higher uptime expectations (>99%)
- Additional validator client software
- Often runs MEV-Boost for higher rewards

### 3. Archive Node

**What it does:**
- Stores **every historical state** of the blockchain
- Can query any account balance at any past block
- Provides deep historical data access

**Who needs it:**
- Block explorers (Etherscan)
- Research and analytics firms
- DeFi protocols needing historical data
- Tax/accounting services

**Requirements:**
- **Massive storage**: 12-20+ TB and growing
- High-performance disks (NVMe recommended)
- Not needed for normal operations

### 4. Light Client

**What it does:**
- Verifies chain using block headers + cryptographic proofs
- Doesn't download full blocks
- Minimal resource usage

**Who needs it:**
- Wallet applications
- Browser extensions
- Mobile apps
- Resource-constrained environments

**Requirements:**
- Minimal: <1 GB storage
- Cannot serve network or provide full RPC

---

## Software Stack

### Execution Clients (EL)

| Client | Language | Notes | Disk Size (Pruned) |
|--------|----------|-------|-------------------|
| **Geth** | Go | Most popular, well-tested | 450-700 GB |
| **Nethermind** | C# | Fast sync, great for enterprise | 500-800 GB |
| **Erigon** | Go | Efficient storage, archive-friendly | 400-700 GB |
| **Besu** | Java | Enterprise features, permissioned networks | 600-900 GB |
| **Reth** | Rust | New, high performance, actively developing | 400-700 GB |

**Archive sizes:** Geth >12TB, Nethermind ~8-12TB, Erigon ~2.5-6TB, Besu ~10-15TB

## Database Architectures

Each execution client uses different database technologies to store blockchain data, which significantly impacts performance, disk usage, and sync times.

**Geth: LevelDB or Pebble**
- **LevelDB** (traditional): Google's key-value store with LSM-tree architecture
  - Battle-tested since Ethereum's launch
  - Good write performance, slower random reads
  - Single-threaded compaction can cause I/O spikes
- **Pebble** (newer): Modern RocksDB-inspired alternative
  - Better concurrent compaction, ~10-15% faster
  - Improved write amplification
  - Becoming the recommended default

**Nethermind: RocksDB**
- High-performance key-value store (Facebook/Meta)
- Multi-threaded compaction (better than LevelDB)
- Highly configurable for enterprise use cases
- Uses separate column families for different data types
- Excellent for high-load RPC services

**Erigon: MDBX (Revolutionary Approach)**
- **Game changer**: Flat storage model instead of traditional trie storage
- MDBX = Modified Memory-Mapped Database (based on LMDB)
- Memory-mapped files for ultra-fast zero-copy reads
- **Key innovation**: Doesn't store Merkle tries on disk—reconstructs them on-the-fly
- Stores account state in flat tables with temporal history tracking
- **Why so efficient?** Eliminates redundant trie node storage, better compression
- **Best for archive nodes**: 2.5-6 TB vs 12+ TB for other clients
- Requires more RAM due to memory mapping

**Besu: RocksDB or Bonsai Tries**
- Standard mode: RocksDB with traditional forest-of-tries storage
- **Bonsai Tries** (newer): Flat key-value storage with on-demand trie generation
  - Dramatically reduces disk usage (similar to Erigon approach)
  - More efficient than traditional trie storage
- Enterprise features: private transactions, permissioned networks

**Reth: MDBX**
- Uses Erigon's proven database architecture
- Written in Rust for memory safety and performance
- Modular, database-first design philosophy
- Expected to match or exceed Erigon's efficiency
- Optimized for modern NVMe SSDs

**Storage Philosophy Comparison:**

| Approach | Clients | How It Works | Pros | Cons |
|----------|---------|--------------|------|------|
| **Trie-Based** | Geth, Nethermind, Besu (default) | Stores Merkle Patricia Trie nodes directly on disk | Straightforward, cryptographically verifiable | Higher disk usage, slower queries |
| **Flat Storage** | Erigon, Reth, Besu (Bonsai) | Stores raw key-value pairs, reconstructs tries when needed | 50-70% less disk space, faster for archive | Requires computation for proof generation |

**Practical Implications:**
- **For validators**: All databases work fine; choose based on stability vs efficiency
- **For RPC nodes**: Nethermind (throughput) or Geth (compatibility)
- **For archive nodes**: Erigon is the clear winner (2.5-6 TB vs 12+ TB)
- **For limited disk space**: Erigon, Reth, or Besu with Bonsai Tries

### Consensus Clients (CL)

| Client | Language | Notes | Disk Size |
|--------|----------|-------|-----------|
| **Lighthouse** | Rust | Fast, low resource usage | 100-220 GB |
| **Prysm** | Go | Popular, feature-rich | 150-300 GB |
| **Teku** | Java | Enterprise-grade, good documentation | 180-350 GB |
| **Nimbus** | Nim | Very lightweight, great for low-spec hardware | 100-220 GB |
| **Lodestar** | TypeScript | Easy for JS developers | 150-300 GB |

**Best Practice:** Run a **minority client** for client diversity (strengthens the network against bugs).

### Additional Components

**Validator Client**
- Usually bundled with your CL (e.g., Lighthouse validator, Prysm validator)
- Manages validator keys
- Signs blocks and attestations
- Alternative: External signers like Web3Signer (better key management)

**MEV-Boost** (Optional for Validators)
- Connects validators to block builder relays
- Allows validators to receive higher-value blocks
- Can increase proposer rewards by 2-5x
- Size: ~15-30 MB, negligible disk usage

**Monitoring Tools** (Highly Recommended)
- **Prometheus**: Metrics collection
- **Grafana**: Visualization dashboards
- **Node exporters**: System metrics
- **Beaconcha.in**: Track validator performance online

---

## Hardware & Network Requirements

### Minimum Hardware (Full Node)

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 4 cores | 6+ cores (8+ for validators) |
| **RAM** | 16 GB | 32 GB |
| **Storage** | 2 TB NVMe SSD | 4 TB NVMe SSD |
| **Bandwidth** | 25 Mbps up/down | 100+ Mbps |
| **Data Transfer** | ~1-2 TB/month | Unlimited plan |

**Critical Notes:**
- **SSD Required**: HDD is too slow, will fail to sync
- **NVMe Preferred**: Much faster than SATA SSD
- **Growing Storage**: Plan for ~50-100 GB growth per month

### Validator-Specific Hardware

| Component | Requirement | Why |
|-----------|-------------|-----|
| **CPU** | 8+ cores | Handle both EL+CL+validator + MEV-Boost |
| **RAM** | 32 GB | Memory-intensive with all components |
| **Uptime** | >99% | Downtime causes missed rewards & penalties |
| **Backup Power** | UPS recommended | Prevent corruption during power loss |
| **Internet** | Stable, low-latency | Timely attestations are critical |

### Network Configuration

**Required Ports (Firewall Rules):**

| Port | Protocol | Purpose | Client |
|------|----------|---------|--------|
| 30303 | TCP/UDP | P2P networking | Geth (EL) |
| 9000 | TCP/UDP | P2P networking | Lighthouse (CL) |
| 8545 | TCP | JSON-RPC (optional, localhost only) | EL API |
| 8551 | TCP | Engine API (EL↔CL communication) | Internal |

**Security:**
- Only open P2P ports to the internet
- Keep RPC ports (8545, 8546) on localhost or behind firewall
- Use reverse proxy (Nginx) if exposing RPC externally
- Never expose validator keys or signing API

**Bandwidth:**
- Ongoing: 1-2 TB/month typical
- Initial sync: 500GB-1TB download
- Validator: Slightly higher (broadcasting blocks/attestations)

---

## Validator Operations

### Getting Started

**Step 1: Acquire 32 ETH**
- Solo staking requires exactly 32 ETH per validator
- Can run multiple validators (64 ETH = 2 validators, etc.)
- Alternative: Join a staking pool (Rocket Pool, Lido) with less ETH

**Step 2: Generate Validator Keys**
- Use the official **Ethereum Staking Deposit CLI**
- Creates validator keypair and deposit data
- **CRITICAL**: Store mnemonic safely offline (lose it = lose access)

**Step 3: Run EL + CL**
- Sync both clients fully (can take 1-3 days)
- Verify sync before proceeding

**Step 4: Deposit 32 ETH**
- Send ETH to the deposit contract using your deposit data
- Transaction is irreversible
- Funds are locked until withdrawals are processed

**Step 5: Start Validator Client**
- Import validator keys
- Wait for activation (depends on validator queue, usually hours to days)

### Validator Responsibilities

**When Proposing:**
- Select transactions from mempool
- Build valid block execution payload
- Broadcast block within slot time
- Typical: ~1-2 proposals per week per validator

**When Attesting:**
- Verify current head block
- Vote on source and target checkpoints
- Broadcast attestation within deadline
- Typical: 1 attestation every ~6.4 minutes

**Sync Committee (Rare):**
- Randomly selected to help light clients
- ~1-2% chance per epoch
- Light duty, small additional rewards

### Sync Modes & Initial Sync Time

| Sync Mode | Time | Disk I/O | Use Case |
|-----------|------|----------|----------|
| **Snap Sync** (Geth) | 4-12 hours | Medium | Default, fast |
| **Checkpoint Sync** (CL) | 5-15 minutes | Low | Start validating quickly |
| **Full Sync** | 3-7 days | Very High | Maximum verification |
| **Archive Sync** | 1-3 weeks | Extreme | Historical data |

**Best Practice for Validators:**
1. Use **checkpoint sync** on CL (near-instant)
2. Use **snap sync** on EL (few hours)
3. Start validating same day

---

## Economics & Rewards

### Validator Rewards

**Base Rewards (Staking Yield):**
- **~3-4% APR** from attestations and proposals
- Higher when network participation is lower
- Paid continuously in ETH
- Formula depends on total staked ETH

**Additional Income Sources:**

1. **Priority Fees (Tips)**
   - Users pay extra to speed up transactions
   - Goes directly to block proposer
   - Variable (congestion-dependent)

2. **MEV (Maximal Extractable Value)**
   - Additional profit from transaction ordering
   - Captured via MEV-Boost + relays
   - Can add 20-50% to proposer rewards
   - Controversial but widely used

**Realistic Annual Returns:**
- Base staking: ~3.5% APR
- With MEV-Boost: ~4.5-5.5% APR
- During high activity: 6-8% APR
- Returns in ETH, not fiat (price volatility applies)

### Penalties

**Inactivity Penalties:**
- Miss attestations: Small penalty (~= missed reward)
- Extended downtime: Gradual leak of stake
- Goal: Incentivize high uptime

**Slashing (Severe):**
Occurs for provably malicious behavior:
- **Double Proposal**: Proposing two different blocks in same slot
- **Surround Vote**: Contradictory attestations
- **Double Vote**: Two different attestations for same slot

**Slashing Consequences:**
- Immediate penalty: ~1 ETH (can be higher)
- Forced exit from validator set
- Additional correlation penalty if many validators slashed simultaneously
- Irreversible damage to reputation

**How to Avoid Slashing:**
- Never run the same validator keys on multiple machines simultaneously
- Use proper backup/failover procedures
- Use slashing protection databases
- Consider distributed validator technology (DVT)

### Withdrawals

**How It Works:**
- Partial withdrawals (rewards): Automatic, ~every 4-5 days
- Full withdrawals (exit): Request via validator, wait ~27 hours + queue
- Funds go to withdrawal address set during deposit
- Withdrawal address can be changed (with restrictions)

**Exit Queue:**
- Limited number of validators can exit per day
- Prevents mass exits during instability
- Longer wait times during high exit demand
  
---

## Common Questions

### General

**Q: What is a block?**  
A: A batch of transactions linked to the previous block; new blocks arrive ~every 12 seconds.

**Q: Mining ended—are blocks still created?**  
A: Yes! Blocks still come every ~12 seconds. Only the selection method changed from mining (PoW) to proposing+attesting (PoS).

**Q: Do I need both EL and CL?**  
A: Yes, for full nodes and validators. The EL executes transactions and stores state; the CL handles PoS consensus and finality.

**Q: Can I run a node without 32 ETH?**  
A: Yes! You can run a full node without any ETH. To become a validator, you need 32 ETH for solo staking, or you can join a staking pool with less.

### Validator-Specific

**Q: What is an attestation?**  
A: A validator's vote confirming the chain head and correct progression; many votes drive finality.

**Q: What is proposing?**  
A: When selected (randomly), a validator builds and publishes the next block's payload.

**Q: How often do I propose blocks?**  
A: With ~1 million validators, you'll propose roughly once every 2 months per validator.

**Q: What happens if I'm offline?**  
A: You'll miss attestation rewards and receive small penalties. Extended downtime increases penalties (inactivity leak).

**Q: Can I run a validator on my laptop?**  
A: Not recommended. Laptops have unreliable uptime, limited cooling, and can sleep/suspend. Use a dedicated server or VPS.

**Q: Can I use a VPS (cloud server)?**  
A: Yes, but be aware of centralization risks (many validators on AWS/Hetzner). Bare metal at home is more decentralized but requires reliability.

### Technical

**Q: How long does initial sync take?**  
A: With snap sync (EL) + checkpoint sync (CL): 4-12 hours. Full sync: 3-7 days.

**Q: Can I pause and resume syncing?**  
A: Yes, clients save progress. You can stop and restart without losing sync state.

**Q: What is the difference between pruned and archive?**  
A: Pruned keeps recent state (last ~128 blocks), much smaller. Archive keeps all historical states, 10-50x larger.

**Q: Do I need a static IP address?**  
A: No, but helpful for port forwarding and peer connections.

**Q: Can I change clients after syncing?**  
A: Yes! You can switch EL or CL clients. You'll need to resync the new client, but it's safe.

### Economics

**Q: How much can I earn?**  
A: ~3.5-5.5% APR in ETH. Varies with network activity, MEV, and uptime.

**Q: When do I get paid?**  
A: Rewards accumulate on the beacon chain. Partial withdrawals happen automatically every 4-5 days.

**Q: What are gas fees?**  
A: Payments users make to have transactions processed. Higher fees = faster processing. Paid to validators (you!).

**Q: What is MEV?**  
A: Maximal Extractable Value - additional profit from ordering transactions optimally (e.g., arbitrage, liquidations).

---

## Quick Reference

### Glossary

**EL (Execution Client)**: Software that executes transactions, runs EVM, maintains state, provides JSON-RPC.

**CL (Consensus Client)**: Software that handles PoS logic, committees, attestations, and finality.

**Validator**: A staker performing propose/attest duties with 32 staked ETH.

**Attestation**: A validator's vote on the current chain head and checkpoint progression.

**Slashing**: Penalty for provably malicious behavior (double signing); can lose stake.

**MEV-Boost**: Tool to connect validators with block builders for higher proposer revenue.

**Pruned Node**: Stores recent state only (~500-900 GB).

**Archive Node**: Stores all historical state (~10-20 TB+).

**Epoch**: 32 slots (~6.4 minutes); used for committee rotation and finality.

**Slot**: 12-second interval; one validator proposes, many attest.

**Finality**: When ⅔ of stake agrees, checkpoints become irreversible (~12-19 minutes).

**DVT (Distributed Validator Technology)**: Splits validator key across multiple nodes for resilience.

**Gas**: Unit measuring computational work; users pay gas fees to validators.

**Gwei**: 0.000000001 ETH; common unit for gas prices.

**EVM (Ethereum Virtual Machine)**: The "computer" that runs smart contracts.

**Smart Contract**: Immutable program running on Ethereum.

**Mempool**: Pool of pending transactions waiting for inclusion in blocks.

**JSON-RPC**: API that applications use to interact with your node.

**LevelDB**: Google's key-value database using LSM-tree architecture; used by Geth (traditional option).

**Pebble**: Modern key-value database inspired by RocksDB; Geth's recommended database option.

**RocksDB**: High-performance key-value store by Meta/Facebook; used by Nethermind and Besu.

**MDBX**: Modified Memory-Mapped Database; used by Erigon and Reth for ultra-efficient storage.

**Trie-Based Storage**: Traditional approach storing Merkle Patricia Trie nodes directly on disk.

**Flat Storage**: Modern approach storing raw key-value pairs, reconstructing tries on-demand; 50-70% more efficient.

**Bonsai Tries**: Besu's flat storage implementation with on-demand trie generation.
