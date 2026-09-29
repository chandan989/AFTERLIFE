# AFTERLIFE

## Permissionless DeFi Exit Compiler

> **The frontend is dead. Your capital isn't.**

AFTERLIFE reconstructs, verifies, and executes exit paths for stranded DeFi positions when the original frontend, API, indexer, or development team is gone.

A user connects a wallet.

AFTERLIFE discovers the contracts behind the position, reconstructs how the assets are nested, compiles the required withdrawal sequence, verifies the exact contracts being called, simulates the outcome, and gives the user a safe recovery transaction.

No cooperation from the failed protocol is required.

---

# The Problem

DeFi promises permissionless ownership.

But leaving a protocol often still depends on infrastructure controlled by the team that built it.

A position may actually look like:

```text
USDG
  ↓ deposit

Vault
  ↓ receive

Vault Share
  ↓ stake

Gauge
  ↓ receive

Staked Receipt Token
```

If the frontend disappears, the user's assets have not necessarily disappeared.

What disappears is the map.

The user now needs to determine:

```text
Which contract holds my position?

Is it a proxy?

What implementation is behind it?

What does this receipt token represent?

Do I need to unstake first?

What function releases the vault shares?

What asset should I receive?

Has the contract changed since the instructions were written?

Can these operations be executed safely?
```

Protocols including DeFi.money and Minterest have already shut down their interfaces while leaving users able to withdraw through direct smart-contract interaction.

AFTERLIFE turns that manual recovery process into software.

---

# The Idea

## DeFi positions should be self-exitable.

AFTERLIFE treats a DeFi position as a dependency graph rather than as a token balance.

Example:

```text
gpUSDG
   │
   │ withdraw()
   ▼
 pUSDG
   │
   │ redeem()
   ▼
 USDG
```

AFTERLIFE determines that graph from permissionless chain data and compiles it into an executable recovery plan.

The result is an:

# EXIT PROOF

An Exit Proof is a machine-checkable description of how a position can currently be recovered.

```text
EXIT PROOF #8F31

Owner
0x71A...

Chain
Arbitrum Sepolia

Position
1,024.84 gpUSDG

Exit Graph

gpUSDG
  ↓ withdraw

pUSDG
  ↓ redeem

USDG

Targets
2 contracts

Contract fingerprints
✓ matched

Proxy implementations
✓ resolved

Recipient
0x71A...

Minimum returned
1,018.00 USDG

Simulation
✓ success

Status

READY TO RECOVER
```

The Exit Proof binds the recovery operation to the exact contracts and constraints that were analyzed.

If the relevant contract implementation changes, the old proof becomes invalid.

---

# How AFTERLIFE Works

```text
                     USER WALLET
                          │
                          ▼
                ┌─────────────────┐
                │ POSITION SCANNER│
                └────────┬────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   RECOVERY COMPILER  │
              │                      │
              │ bytecode             │
              │ proxy state          │
              │ logs                 │
              │ balances             │
              │ interface probes     │
              │ view calls           │
              │ historical evidence  │
              └──────────┬───────────┘
                         │
                         ▼
                 EXIT DEPENDENCY
                      GRAPH
                         │
                         ▼
             ┌───────────────────────┐
             │ STYLUS EXIT VERIFIER  │
             │        Rust           │
             │                       │
             │ code fingerprints     │
             │ known patterns        │
             │ operation checks      │
             │ plan constraints      │
             └───────────┬───────────┘
                         │
                         ▼
                    EXIT PROOF
                         │
                         ▼
                    SIMULATION
                         │
                         ▼
               ┌──────────────────┐
               │ RECOVERY EXECUTOR│
               └─────────┬────────┘
                         │
                         ▼
                       USER
                         │
                         ▼
                       USDG
```

---

# 1. Position Scanner

AFTERLIFE begins with the wallet rather than with a protocol frontend.

It identifies potential position assets from:

```text
ERC-20 balances

transaction history

Transfer events

staking receipts

vault shares

known position contracts

proxy relationships
```

For each candidate contract it can query permissionless RPC primitives including:

```text
eth_getCode

eth_getStorageAt

eth_call

eth_getLogs
```

No API belonging to the target protocol is required.

---

# 2. Recovery Compiler

This is the core innovation.

The Recovery Compiler attempts to determine:

```text
TOKEN
  ↓

What issued it?
  ↓

What asset backs it?
  ↓

Is it currently staked?
  ↓

What operation releases it?
  ↓

Does another position sit underneath it?
```

Instead of blindly searching for a `withdraw()` selector, the compiler combines multiple forms of evidence.

### Interface probing

Examples:

```text
asset()

totalAssets()

convertToAssets()

previewRedeem()

stakingToken()

earned()

balanceOf()

allowance()
```

### Known standards

Initial support focuses on contracts where behavior can be strongly verified:

```text
ERC-20

ERC-4626

ERC-1967 proxies

common staking wrappers
```

### Bytecode analysis

Runtime bytecode is inspected for known operation patterns and selectors.

### Event analysis

Historical events help identify relationships such as:

```text
deposit

stake

withdraw

redeem

transfer
```

### View-call validation

Candidate relationships are checked against live contract state.

---

# 3. Exit Dependency Graph

The compiler converts discovered relationships into a graph.

Example:

```text
                ┌──────────────┐
                │ gpUSDG       │
                └──────┬───────┘
                       │
                    unstake
                       │
                       ▼
                ┌──────────────┐
                │ pUSDG        │
                └──────┬───────┘
                       │
                     redeem
                       │
                       ▼
                ┌──────────────┐
                │ USDG         │
                └──────────────┘
```

Each edge represents a real operation.

Each node represents a real asset or contract state.

A recovery plan is therefore not:

```text
call withdraw()
```

It is:

```text
Position A
    ↓ operation X

Position B
    ↓ operation Y

Underlying Asset
```

---

# 4. Stylus Exit Verifier

AFTERLIFE uses Arbitrum Stylus for the compute-heavy verification layer.

The verifier is written in Rust.

Stylus is suited to this role because bytecode processing and larger in-memory workloads are significantly cheaper than equivalent EVM computation for many workloads. Arbitrum documents compute as commonly 10–100× cheaper and memory as 100–500× cheaper depending on the workload.

The verifier performs operations including:

```text
external bytecode inspection

EXTCODEHASH-style fingerprinting

selector scanning

supported-pattern matching

recovery-plan validation

target validation

plan hashing
```

The goal is not to pretend Rust can magically understand arbitrary contracts.

The goal is to make the expensive deterministic verification step efficient enough to perform onchain.

---

# 5. Exit Proof

The output of verification is an Exit Proof.

Conceptually:

```solidity
struct ExitProof {
    address owner;

    uint256 chainId;

    bytes32 graphHash;

    bytes32[] targetCodeHashes;

    address finalAsset;

    address recipient;

    uint256 minimumAmountOut;

    uint256 deadline;
}
```

The proof commits to:

```text
WHO

owns the position

WHERE

the recovery is happening

WHAT

contracts may be called

WHICH VERSION

of those contracts was analyzed

HOW

the position will be unwound

WHERE

the recovered assets will go

HOW MUCH

must be recovered at minimum
```

Changing any important part produces a different proof.

---

# 6. Simulation

An Exit Proof is not immediately executed.

The entire recovery path is first simulated against current state.

AFTERLIFE checks:

```text
all calls succeed

expected assets leave the position

expected underlying assets arrive

recipient is correct

minimum output is satisfied

the position is actually reduced

unexpected assets are not transferred
```

Example:

```text
RECOVERY SIMULATION

Before

gpUSDG       1,024.84
pUSDG            0.00
USDG             7.42

After

gpUSDG           0.00
pUSDG            0.00
USDG         1,032.26

Recovered

1,024.84 USDG

Result

✓ SAFE TO EXECUTE
```

---

# 7. Recovery Executor

Only verified recovery plans reach execution.

Depending on the target protocol, AFTERLIFE can use:

```text
direct user calls

safe sequential execution

compatible batch execution

router-based execution
```

Atomic execution is used where the target contracts permit it.

AFTERLIFE does not pretend every protocol can be atomically unwound through a generic router.

If three user-originated transactions are genuinely required:

```text
1. UNSTAKE

2. REDEEM

3. WITHDRAW
```

AFTERLIFE executes and tracks those three steps rather than hiding unsafe assumptions behind a "one click" claim.

---

# Jev

Jev sits inside discovery, not execution.

Its job is to help answer questions such as:

```text
What might this unknown receipt token represent?

Which detected methods are likely related?

What contract appears to issue the underlying shares?

Why did a candidate recovery path revert?
```

Jev can produce:

```text
candidate classifications

candidate dependencies

function interpretations

human-readable explanations
```

Jev cannot approve a recovery.

Jev cannot bypass verification.

Jev cannot move funds.

The model proposes.

The verifier proves what can actually be checked.

The user signs.

---

# Confidence Levels

Not every protocol can be safely reconstructed.

AFTERLIFE makes uncertainty visible.

### VERIFIED

```text
✓ supported structure

✓ contracts fingerprinted

✓ required calls identified

✓ simulation passed

✓ output constraints satisfied

EXECUTION ENABLED
```

### PARTIALLY VERIFIED

```text
✓ likely exit route

✓ simulation passed

! unsupported structural element

MANUAL CONFIRMATION REQUIRED
```

### UNKNOWN

```text
Possible exit discovered.

AFTERLIFE cannot establish
sufficient confidence for
automatic execution.

EXECUTION DISABLED
```

A safe refusal is a valid result.

---

# What Makes AFTERLIFE Different

## Block explorers

Arbiscan can expose a contract method.

It cannot tell a normal user:

```text
what their position represents

what must be withdrawn first

how several contracts depend on each other

what the final asset should be

whether an implementation changed

whether the complete exit succeeds
```

AFTERLIFE reconstructs the position.

---

## Protocol-specific unwind tools

Existing DeFi products can provide sophisticated one-transaction exits for protocols they explicitly support.

AFTERLIFE solves a different problem:

> **What happens when the application you're relying on no longer exists?**

The target protocol does not need to integrate AFTERLIFE.

---

## Emergency guardians

AFTERLIFE is not another risk-monitoring or exploit-detection bot.

It does not attempt to predict whether a protocol is dying.

Its job begins with a simpler question:

> **Can this wallet still get out?**

---

# USDG Integration

The hackathon build uses Paxos USDG as the underlying asset in the recovery demonstration.

Paxos currently publishes USDG on Arbitrum Sepolia at:

```text
0xFFC95faa3d63Cde504a05B567C600B78C0b41892
```


USDG is not added as a decorative transfer at the end of the demo.

It is the asset being deposited, wrapped into a DeFi position, stranded after the demo protocol dies, and ultimately recovered by AFTERLIFE.

---

# Phantom Protocol

The demo includes a miniature DeFi application called:

# PHANTOM

Its structure is:

```text
USDG
 │
 │ deposit
 ▼
PhantomVault
 │
 │ ERC-4626 shares
 ▼
pUSDG
 │
 │ stake
 ▼
PhantomGauge
 │
 │ receipt position
 ▼
gpUSDG
```

The normal PHANTOM frontend knows this structure.

AFTERLIFE does not receive that information directly.

---

# The Kill Test

The demo is designed around one visual moment.

### Step 1

Deposit:

```text
1,000 USDG
```

into PHANTOM.

Stake the vault shares.

---

### Step 2

Kill PHANTOM.

```text
PHANTOM

404

SERVICE UNAVAILABLE
```

Stop:

```text
frontend

application API

indexer
```

The smart contracts remain deployed.

---

### Step 3

Open AFTERLIFE.

Connect the same wallet.

```text
SCANNING WALLET...

POSITION FOUND

1,0XX gpUSDG
```

---

### Step 4

AFTERLIFE reconstructs:

```text
gpUSDG
  ↓

PhantomGauge
  ↓ withdraw

pUSDG
  ↓

PhantomVault
  ↓ redeem

USDG
```

No PHANTOM frontend or API is used.

---

### Step 5

AFTERLIFE generates:

```text
EXIT PROOF

Code fingerprints       ✓

Vault structure         ✓

Underlying              USDG

Simulation              ✓

Expected recovery       1,0XX USDG

Minimum recovery        1,0XX USDG

Recipient               0xUSER

READY
```

---

### Step 6

Click:

# RECOVER

Sign the recovery.

---

### Step 7

Show the result.

```text
RECOVERY COMPLETE

1,0XX USDG

returned to

0xUSER
```

Then show:

```text
PHANTOM FRONTEND

OFFLINE
```

And finally open the chain explorer to prove the recovery occurred completely through the deployed contracts.

---

# The Demo Line

> **We didn't restore the dead app.**

> **We reconstructed the exit from the blockchain.**

---

# Recovery Readiness

The exact same engine also works before an emergency.

A user can scan current positions and ask:

# “If this frontend disappeared tonight, could I still get out?”

Example:

```text
RECOVERY READINESS

Aave position
VERIFIED

USDG Vault
VERIFIED

Unknown Farm
PARTIAL

Legacy LP
UNKNOWN
```

This gives AFTERLIFE utility before a shutdown happens without requiring a separate monitoring product.

It is simply another use of the same Exit Compiler.

---

# Safety Invariants

Every executable recovery follows strict rules.

### Owner binding

The position owner is part of the Exit Proof.

### Recipient binding

Recovered assets can only go to the approved recipient.

### Code fingerprinting

The verifier checks analyzed contract versions.

```text
expectedCodeHash
        ==
currentCodeHash
```

If not:

```text
CONTRACT CHANGED

EXIT PROOF INVALID
```

### Minimum output

The transaction reverts if recovery produces less than the configured floor.

### Expiration

Old Exit Proofs expire.

### Explicit targets

Recovery execution cannot introduce arbitrary unverified contracts.

### No AI execution authority

Model output never overrides deterministic checks.

### Simulation requirement

Executable plans must first simulate successfully.

---

# Supported MVP Recovery Classes

The hackathon build deliberately supports a narrow but real subset.

```text
ERC-20 assets

ERC-4626 vaults

ERC-1967 proxy detection

common staking wrappers

nested:

staking receipt
    ↓

vault share
    ↓

underlying token
```

That is sufficient to demonstrate the new primitive without pretending to solve every DeFi protocol in existence.

---

# Smart Contracts

```text
AfterlifeVerifier.rs
```

Stylus/Rust verification engine.

Responsibilities:

```text
bytecode inspection

code fingerprinting

selector matching

pattern validation

Exit Proof verification
```

---

```text
AfterlifeExecutor.sol
```

Constrained recovery executor.

Responsibilities:

```text
approved targets

approved operations

recipient enforcement

minimum-output enforcement

deadline enforcement

asset accounting
```

---

```text
PhantomVault.sol
```

ERC-4626 USDG vault used for the kill test.

---

```text
PhantomGauge.sol
```

Secondary staking layer.

---

# Offchain Recovery Compiler

```text
scanner/

proxyResolver/

interfaceProber/

eventAnalyzer/

graphCompiler/

simulator/

jev/
```

This portion performs discovery that cannot or should not happen inside the execution contract.

The final execution remains constrained by onchain verification.

---

# MVP Repository

```text
afterlife/
│
├── apps/
│   └── web/
│
├── compiler/
│   ├── scanner.ts
│   ├── proxy.ts
│   ├── interfaces.ts
│   ├── events.ts
│   ├── graph.ts
│   ├── simulation.ts
│   └── jev.ts
│
├── stylus/
│   └── verifier/
│       └── src/
│           ├── lib.rs
│           ├── fingerprint.rs
│           ├── selectors.rs
│           └── proof.rs
│
├── contracts/
│   ├── AfterlifeExecutor.sol
│   ├── ExitProof.sol
│   ├── PhantomVault.sol
│   └── PhantomGauge.sol
│
├── test/
│
└── README.md
```

---

# Hackathon Scope

## Ship

```text
✓ wallet scanner

✓ ERC-4626 discovery

✓ nested staking discovery

✓ ERC-1967 proxy resolution

✓ Exit Dependency Graph

✓ Stylus verifier

✓ Exit Proof

✓ simulation

✓ constrained execution

✓ USDG integration

✓ PHANTOM kill test
```

## Do Not Waste Time On

```text
✗ cross-chain recovery

✗ universal bytecode decompilation

✗ DAO governance

✗ tokens

✗ protocol scoring

✗ yield discovery

✗ DNS monitoring

✗ autonomous keepers

✗ mobile app

✗ giant adapter library

✗ generic AI chatbot
```

---

# Why Arbitrum

AFTERLIFE uses Arbitrum for more than cheap transactions.

The Recovery Verifier performs compute- and memory-heavy contract analysis using Stylus.

The execution contracts and USDG test position live on Arbitrum.

The entire demonstration therefore depends on:

```text
Arbitrum smart contracts

+

Stylus computation

+

Arbitrum transaction execution

+

USDG on Arbitrum
```

The chain is part of the product architecture rather than simply the place where a token was deployed.

---

# One-Sentence Pitch

> **AFTERLIFE is a permissionless exit compiler that reconstructs a stranded DeFi position from onchain state, generates a verified Exit Proof, and returns the recoverable assets to the owner—even after the original application disappears.**

---

# 10-Second Pitch

> **Your DeFi protocol disappeared. AFTERLIFE doesn't need its frontend, API, or developers. It reads the contracts, reconstructs your position, proves the exit, and gets your assets back.**

---

# The Thesis

```text
If the smart contract survives,

the exit should survive with it.
```

# AFTERLIFE

### **The frontend is dead. Your capital isn't.**