
# Solidity Compiler Internals for Auditors

## Compilation Pipeline Overview
- Solidity compilation is not “Solidity → EVM” directly.
- It goes through multiple stages:

Solidity Source
   ↓
AST (Abstract Syntax Tree)
   ↓
IR (Intermediate Representation - Yul IR)
   ↓
Yul Optimizer
   ↓
EVM Bytecode

## Why Auditors Must Care
- Optimizer changes logic
- Storage layout determined before bytecode
- IR stage can introduce subtle behavior
- Inline assembly bypasses safety checks

## AST (Abstract Syntax Tree)
- AST represents parsed Solidity code structure.
- Audit Importance
- Tools like:
    - Slither
    - Mythril
    - Solhint
    - operate on AST.

### Auditor Should Check:
- Is logic dependent on AST rewriting?
- Are modifiers inlined?
- Are inherited functions resolved correctly?

## IR (Intermediate Representation – Yul)
- Modern Solidity (≥0.6+) compiles to Yul IR before bytecode.
- Yul is low-level but structured.

Example Solidity:
```
x += 1;
```

Becomes Yul:
```
let tmp := sload(slot)
sstore(slot, add(tmp, 1))
```
### Why This Matters
- Optimizer operates at IR level
- Incorrect assumptions about ordering can be dangerous
- Inline assembly merges into IR

## Optimizer Behavior
- Compiler flag:
```
--optimize
--optimize-runs
```
Optimizer Can:
- Reorder instructions
- Remove dead code
- Inline functions
- Combine storage reads
- Eliminate checks

### Auditor Checklist
- Does optimizer change logic?

Edge case:
```
if (false) {
   dangerous();
}
```

Optimizer removes block.

- Reentrancy guard optimized away?
    - Rare but possible if incorrectly structured.
- Variable caching
    - Optimizer caches storage reads:
```
uint tmp = x;
```

If x changed via delegatecall → inconsistency.

## Storage Layout Generation

Storage layout determined at compile time.

### Rules:
- Slot 0 → first state variable
- Packed if possible
- Inheritance merged top-down (C3 linearization)

Example:
```
uint128 a;
uint128 b;
```

Both stored in same slot.

### Upgradeable Risk

Changing:
```
uint128 a;
uint128 b;
```

To:
```
uint256 a;
uint128 b;
```

Shifts storage slots → catastrophic corruption.

Auditor Must Use
- solc --storage-layout

To verify layout consistency.

## Function Selector Generation
- Each external/public function:
    - bytes4(keccak256("transfer(address,uint256)"))
- First 4 bytes = function selector.

### Collision Risk
- Two functions can collide (rare but possible).
- Example historical collision attack.
- Auditor Must Check
- Any fallback dispatch?
- Proxy manual selector handling?
- Custom function routing?

## ABI Encoding & Decoding
- Compiler generates:
- calldata decoding
- emory allocation
- return data encoding

### Audit Risk
- Improper decoding in assembly:
```
calldataload(4)
```

If parameters not validated → exploit possible.

## Memory Model Internals

Solidity memory layout:
```
0x00 - 0x3f  scratch space
0x40         free memory pointer
0x60         zero slot
```

Free memory pointer grows upward.

### Audit Importance
- Dirty memory reuse?
- Incorrect assembly memory handling?
- Free pointer not updated?

## Call Frame & Stack Depth
- EVM stack max depth = 1024.
- Each function call adds stack items.

### Audit Risk
- Deep recursion?
- Complex nested calls?
- Inline assembly stack misuse?

## Error Handling Compilation

require() compiles to:
```
if (!condition) revert(...)
```

- Custom errors cheaper than revert strings.
- Gas Insight
    - Revert string → stored in bytecode → increases deployment size.
- Custom errors:
    - Smaller bytecode
    - Cheaper runtime
    
### Auditor checks:
- Large revert strings causing deployment risk?

## Delegatecall Compilation
delegatecall(gas(), addr, ...)

Compiler emits raw EVM delegatecall.

### Critical:
- Storage slot remains same.
- Auditor Must Understand
- Delegatecall executes:
   - callee code
   - caller storage
   - caller msg.sender
   - caller msg.value
- This is why proxy takeover is possible.

## Constructor Compilation
- Constructor code stored separately.
- Deployment bytecode:
- Constructor runs
- Returns runtime bytecode
- Runtime bytecode excludes constructor.

### Audit Risk
- In proxy:
    - Constructor never runs
    - Must use initializer

## Immutable Variable Handling
- Immutable variables are:
    - Stored in bytecode
    - Not in storage
- Compiler inserts them at runtime via codecopy.

### Audit Insight
- Immutable:
    - Cannot be modified
    - Cheaper than storage
- Not usable in proxy (since stored in implementation bytecode)

## Constants Handling
- Constants replaced at compile time.

Example:
```
uint constant FEE = 10;
```
Compiler inlines 10.

## Fallback Dispatch Table

Compiler builds dispatcher:
```
if selector == X → jump
if selector == Y → jump
else → fallback
```

### Auditor must understand this when reviewing:
- Manual fallback routing
- Diamond proxies
- Function selector shadowing

## Inline Assembly Integration
- Assembly merges directly into IR.
    - No safety checks:
    - No overflow checks
    - No bounds checks
    - No type safety

### Auditor Must Check
- sstore slot correct?
- calldataload offset correct?
- memory pointer updated?
- Return data size correct?

## Compiler Bugs
- Historically:
    - ABI encoder bugs
    - Optimizer miscompilation
    - Storage corruption bugs

### Auditor must check:
```
solc --version
```
And cross-reference:
https://docs.soliditylang.org/en/latest/bugs.html

## Optimizer Runs Parameter

Example:
```
--optimize-runs=200
```
- Low runs:
    - Cheaper deployment
    - More expensive execution
- High runs:
    - Expensive deployment
    - Cheaper repeated calls

### Audit implication:
- Gas assumptions may differ between test and production.

## Bytecode Structure

Deployment bytecode:
```
[constructor code][runtime return]
```

Runtime bytecode:
```
[dispatcher][function bodies][metadata]
```

## Metadata includes:
- Compiler version
- IPFS hash
- Auditor can verify source match via metadata hash.

### Metadata & Source Verification
-  Compiler appends CBOR metadata.

### Auditor can:
-  Extract compiler version
- Verify exact compilation settings
-  If mismatch → possible malicious deployment.

# Most Critical Compiler-Level Knowledge for Auditors
- Master these deeply:
- Storage layout generation
- delegatecall mechanics
- Function selector derivation
- Optimizer side effects
- Constructor vs runtime bytecode
- Immutable vs storage difference
- ABI encoding internals
- Memory model (free pointer, scratch space)

# Auditor should

- Read raw bytecode
- Disassemble with evm.codes
- Compare optimized vs non-optimized build
- Inspect storage layout JSON
- Check metadata hash
- Simulate delegatecall at EVM level


