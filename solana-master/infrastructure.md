# Infrastructure: Clusters, Deployment, and Local Development

## Solana Clusters

### Overview

| Cluster | Purpose | Endpoint |
|---------|---------|----------|
| **Mainnet Beta** | Production, real SOL | `https://api.mainnet-beta.solana.com` |
| **Devnet** | Development testing | `https://api.devnet.solana.com` |
| **Testnet** | Stress testing, validator testing | `https://api.testnet.solana.com` |

### Cluster Details

**Mainnet Beta**
- Live production environment
- Real SOL tokens with monetary value
- Gossip: `entrypoint.mainnet-beta.solana.com:8001`

**Devnet**
- Public development and testing
- Free airdrops via faucet
- May experience ledger resets
- Gossip: `entrypoint.devnet.solana.com:8001`

**Testnet**
- Network upgrade stress testing
- Typically runs newer software than devnet/mainnet
- Subject to frequent ledger resets
- Gossip: `entrypoint.testnet.solana.com:8001`

### Rate Limits (All Public Endpoints)

| Limit | Value |
|-------|-------|
| Requests per 10 seconds per IP | 100 |
| Concurrent connections per IP | 40 |

**Warning**: Public RPC endpoints are **not intended for production**. Use private RPC providers (Helius, QuickNode, Triton, etc.) for production apps.

### Configuring Cluster

```bash
# CLI
solana config set --url devnet
solana config set --url mainnet-beta
solana config set --url https://your-rpc-provider.com

# Check current config
solana config get
```

```typescript
// TypeScript
import { Connection, clusterApiUrl } from '@solana/web3.js';

// Using cluster shortcuts
const devnetConnection = new Connection(clusterApiUrl('devnet'));
const mainnetConnection = new Connection(clusterApiUrl('mainnet-beta'));

// Using custom RPC
const connection = new Connection('https://your-rpc-provider.com');
```

---

## Program Deployment

### Build Process

```bash
# Build for Solana BPF target
cargo build-sbf

# Output: target/deploy/<program_name>.so
```

### Deployment Phases

1. **Buffer Initialization**
   - Creates buffer account
   - Establishes deployer's write authority

2. **Buffer Writes**
   - Bytecode split into ~1KB chunks
   - Sent at ~100 TPS to leader's TPU

3. **Finalization**
   - Copies bytecode from buffer to program data account
   - Verifies integrity

### Deploy Commands

```bash
# Deploy new program
solana program deploy target/deploy/my_program.so

# Deploy to specific address (keypair file)
solana program deploy target/deploy/my_program.so --program-id ./program-keypair.json
```

### Handling Network Congestion

```bash
# Increase priority fee
solana program deploy target/deploy/my_program.so \
  --with-compute-unit-price 1000

# Increase retry attempts (default: 5)
solana program deploy target/deploy/my_program.so \
  --max-sign-attempts 10

# Use RPC instead of TPU (stake-weighted routing)
solana program deploy target/deploy/my_program.so \
  --use-rpc
```

### Upgrade Authority Management

```bash
# View current authority
solana program show <PROGRAM_ADDRESS>

# Transfer authority
solana program set-upgrade-authority <PROGRAM_ADDRESS> \
  --new-upgrade-authority <NEW_AUTHORITY_PUBKEY>

# Make immutable (IRREVERSIBLE)
solana program set-upgrade-authority <PROGRAM_ADDRESS> --final
```

### Upgrading Programs

```bash
# Deploy upgrade to existing program
solana program deploy target/deploy/my_program.so \
  --program-id <EXISTING_PROGRAM_ADDRESS>
```

### Closing Programs

```bash
# Reclaim SOL (PERMANENT - cannot reuse address)
solana program close <PROGRAM_ADDRESS>
```

---

## Local Validator (solana-test-validator)

### Installation

```bash
# Via Solana Toolkit
npx -y mucho@latest install

# Or via Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

### Starting the Validator

```bash
# Basic start
mucho validator
# or
solana-test-validator

# With reset (fresh ledger)
mucho validator --reset

# Custom ledger location
mucho validator --ledger /path/to/ledger
```

Default RPC endpoint: `http://localhost:8899`

### Connecting CLI

```bash
solana config set --url localhost
solana ping  # Verify connection
```

### Key Advantages

- No RPC rate limits
- Unlimited airdrops
- Instant state resets
- Direct program deployment
- Clone accounts/programs from other clusters
- Configurable epoch length
- Time manipulation

### Cloning from Other Clusters

```bash
# Clone account from devnet
mucho validator --reset \
  --url devnet \
  --clone <ACCOUNT_ADDRESS>

# Clone upgradeable program (includes program data account)
mucho validator --reset \
  --url mainnet-beta \
  --clone-upgradeable-program <PROGRAM_ADDRESS>

# Multiple clones
mucho validator --reset \
  --url mainnet-beta \
  --clone <ACCOUNT_1> \
  --clone <ACCOUNT_2> \
  --clone-upgradeable-program <PROGRAM_ADDRESS>
```

### Loading Accounts from Files

```bash
# Load account from JSON file
mucho validator --account <ADDRESS> /path/to/account.json
```

### Feature Management

```bash
# Check runtime features
solana feature status

# Deactivate specific feature (fresh ledger required)
mucho validator --reset \
  --deactivate-feature <FEATURE_PUBKEY>
```

### Useful Commands

```bash
# Stream logs
solana logs

# Deploy program
solana program deploy target/deploy/my_program.so

# View program details
solana program show <PROGRAM_ADDRESS>

# Get account info
solana account <ADDRESS>

# Airdrop SOL
solana airdrop 10 <ADDRESS>
```

### Version Management

```bash
# Check validator version
mucho validator --version

# Install specific Solana CLI version
solana-install init 1.18.0
```

---

## Program Limitations

### Prohibited Rust Libraries

The following `std` modules are unavailable in on-chain programs:

| Module | Reason |
|--------|--------|
| `rand` | Non-deterministic |
| `std::fs` | No filesystem access |
| `std::net` | No network access |
| `std::future` | No async runtime |
| `std::process` | No process spawning |
| `std::sync` | Single-threaded |
| `std::task` | No async tasks |
| `std::thread` | Single-threaded |
| `std::time` | Non-deterministic |

**Expensive operations to avoid:**
- `bincode` serialization (use Borsh)
- String formatting
- `println!` / `print!` (use `msg!` macro instead)

### Compute Constraints

| Constraint | Limit |
|------------|-------|
| Max compute units | 1,400,000 per transaction |
| Default per instruction | 200,000 CU |
| Max call stack depth | 64 frames |
| Max CPI depth | 4 levels |
| Max accounts per instruction | 64 |
| Max account data size | 10 MB |

### Float Operations

Float operations use software emulation and are expensive:
- Multiply: 176 instructions (vs 8 for integer)
- Divide: 219 instructions (vs 9 for integer)

**Use fixed-point arithmetic instead.**

### Other Limitations

- No static writable data or global variables
- No native signed division (use library functions)
- Program code is shared read-only across parallel executions

---

## RPC API Overview

### Commitment Levels

| Level | Description |
|-------|-------------|
| `finalized` | Supermajority confirmed, max lockout |
| `confirmed` | Supermajority votes received |
| `processed` | Most recent block (may be skipped) |

Default is `finalized` when not specified.

### Key HTTP Methods

| Method | Purpose |
|--------|---------|
| `getAccountInfo` | Fetch account data |
| `getBalance` | Get SOL balance |
| `getLatestBlockhash` | Get recent blockhash for transactions |
| `getTransaction` | Fetch transaction details |
| `getSignatureStatuses` | Check transaction status |
| `sendTransaction` | Submit signed transaction |
| `simulateTransaction` | Simulate without execution |
| `getRecentPrioritizationFees` | Estimate priority fees |
| `getProgramAccounts` | Fetch all accounts owned by program |

### Key WebSocket Subscriptions

| Method | Purpose |
|--------|---------|
| `accountSubscribe` | Monitor account changes |
| `programSubscribe` | Monitor program account changes |
| `logsSubscribe` | Stream transaction logs |
| `signatureSubscribe` | Track signature confirmation |
| `slotSubscribe` | Monitor slot progression |

### WebSocket Connection

```typescript
// Default port: 8900
const ws = new WebSocket('ws://localhost:8900');

// Or derive from HTTP endpoint
const wsEndpoint = httpEndpoint
  .replace('https://', 'wss://')
  .replace('http://', 'ws://');
```

### Response Structure

All RPC responses include context:

```typescript
interface RpcResponse<T> {
  context: {
    slot: number;  // Slot at which data was evaluated
  };
  value: T;
}
```

### Filtering with memcmp

```typescript
const accounts = await connection.getProgramAccounts(programId, {
  filters: [
    {
      memcmp: {
        offset: 0,  // Byte offset
        bytes: base58EncodedData,
      },
    },
    {
      dataSize: 165,  // Exact account size
    },
  ],
});
```
