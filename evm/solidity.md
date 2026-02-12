# Solidity Language Features — Security Auditor Cheat Sheet
## Data Locations (CRITICAL)
- storage
- memory
- calldata

### Why It Matters?

- Storage modification = state change
- Memory = temporary copy
- Calldata = read-only, cheaper, immutable

### Auditor Must Check

- Are storage references unintentionally modifying state?
- Is calldata used for external function parameters?
- Any storage pointer aliasing?

Dangerous Example
```
function update(uint[] storage arr) internal {
    arr[0] = 999; // modifies original state
}
```
### Key Risk

- Storage pointer corruption
- Unexpected state mutation

## Visibility Specifiers
- public
- external
- internal
- private

### Auditor Must Verify
- Should function really be public?
- Internal functions exposed via delegatecall?
- Public state variables auto-generate getters

### Key Insight
- external saves gas for calldata arrays
- private is NOT private on-chain (just restricted access)

## Function Modifiers
modifier onlyOwner() {
    require(msg.sender == owner);
    _;
}

### Audit Focus
- Modifier order matters
- Is _ placed correctly?
- Can modifier be bypassed?
- Modifier with external calls?

### Risk
Improper modifier structure:
```
modifier unsafe() {
    _;
    require(condition); // executes AFTER function
}
```
## State Variables & Storage Layout
### Critical for:
- Upgradeable contracts
- Proxy patterns
- Auditor Must Check
- Variable order consistency
- Inheritance storage layout
- Packing optimization side effects
- Shadowed variables

Storage Packing Example
```
uint128 a;
uint128 b; // packed in same slot
```

### Risk
- Unexpected overwrites if misaligned in upgrade.

## Constructors vs Initializers
### Constructor
- Runs once at deployment.

### Initializer (Upgradeable)
- Must be protected: initializer

### Audit Check
- Can initializer be called twice?
- Is implementation contract initialized?
- Is constructor logic missing in proxy?
## Fallback & Receive
- receive() external payable {}
- fallback() external payable {}

### Auditor Focus
- Is fallback unintentionally payable?
- Can ETH get locked?
- Any logic in fallback?

## Error Handling
- require
  - Reverts and refunds gas
- assert
  - For invariants (uses all gas pre-0.8 behavior difference)
- revert
  - Custom revert
  - Custom Errors (Gas Efficient)
     - error Unauthorized();

### Audit Focus
- assert misused?
- Missing require validation?
- Meaningful revert reasons?

## Low-Level Calls
- call
- delegatecall
- staticcall

delegatecall = MOST DANGEROUS
### Auditor Must Check
- Is return value verified?
- Is delegate target trusted?
- Is storage collision possible?

### Safe Pattern
```
(bool success, bytes memory data) = target.call(payload);
require(success);
```
## msg, tx, block Globals
- msg.sender
- msg.value
- tx.origin
- block.timestamp
- block.number
- blockhash()

### Audit Focus
- Never use tx.origin
- Risky
  - block.timestamp (miner manipulation)
  - block.number for randomness

## Payable & Ether Handling
### Auditor Checks
- Is payable necessary?
- Is msg.value validated?
- Can funds be trapped?
- Forced ETH handling?

## Mappings
mapping(address => uint)

### Important Properties
- No iteration
- Default value = 0
- Cannot detect existence

### Audit Focus
- Missing existence check
- Mapping used for access control?
- Mapping reset logic flawed?

## Arrays
- Dynamic Arrays
- Gas-heavy loops

### Audit Focus
- Unbounded loops
- Deleting array elements correctly?
- Pop vs delete confusion?

## Enums
- Stored as uint internally.

### Audit Focus
- Implicit casting?
- Invalid enum values via low-level calls?

## Structs
### Audit Focus
- Storage vs memory struct copying
- Partial updates
- Nested structs storage references

## Inheritance & Linearization
- Solidity uses C3 linearization.

### Audit Focus
- Function override correctness?
- Virtual / override keywords used?
- Multiple inheritance shadowing?

## Virtual & Override
- Required in Solidity ≥0.6.

- function foo() public virtual {}
- function foo() public override {}

### Audit Check
- Missing override?
- Unexpected base function execution?

## Abstract Contracts & Interfaces
### Audit Focus
- Interface mismatch?
- Return values ignored?
- ERC standard compliance?

## Immutable & Constant
- uint public constant FEE = 10;
- address immutable owner;

### Audit Focus
- Immutable set correctly?
- Constant used properly?
- Upgradeable conflict with immutable?

## Libraries
- library SafeMath {}

### Audit Focus
- Library linked correctly?
- Using for syntax correct?
- Delegatecall via library?

## Events
event Transfer(address indexed from, address indexed to);

### Audit Focus
- Missing critical event?
- Indexed parameters correct?
- Event emitted after state change?

## unchecked Block

```
unchecked {
    i++;
}
```
### Audit Focus
- Overflow risk intentional?
- Used in financial logic?

## Type Casting
- uint8(x)
- address(uint160(x))

### Audit Focus
- Downcasting overflow?
- Address truncation?
- Signed vs unsigned issues?

## ABI Encoding
- abi.encode()
- abi.encodePacked()
- abi.decode()

### Critical Risk
- Hash collision with encodePacked

Dangerous
```
keccak256(abi.encodePacked(a, b));
```

- If variable-length types used → collision possible.

## Selfdestruct
- selfdestruct(payable(addr));

### Audit Focus
- Access restricted?
- Funds drained?
- Forced ETH griefing?

## Assembly (Yul)
```
assembly {
    sstore(slot, value)
}
```
High Risk Area
### Auditor Must:
- Verify storage slot correctness
- Memory pointer safety
- No arbitrary writes
- No dirty memory reuse

## Compiler Version
- pragma solidity ^0.8.20;

### Audit Focus
- Floating pragma?
- Compiler bugs?
- Optimizer settings?

# Solidity global items (built-ins)
## msg (Message Context)

Represents the current call.

### ✅ msg.sender

#### Functionality
- Address that directly called the current function.
- Changes across contract-to-contract calls.
- In delegatecall, remains original caller.

#### Audit Focus
- Used for access control?
- Used inside delegatecall logic?
- Used in constructor (deployment nuance)?

#### Risks
- Phishing via proxy contracts.
- Confusion in delegatecall (msg.sender not proxy).

### ✅ msg.value
#### Functionality
- Amount of ETH (wei) sent with the call.

#### Audit Focus
- Is payable required?
- Is msg.value validated?
- Can extra ETH be trapped?

#### Risks
- Missing validation:
- require(msg.value == price);
- Overpayment exploit
- Locked ETH

### ✅ msg.data
#### Functionality
- Full calldata of the transaction.

#### Audit Focus
- Used in signature hashing?
- Used in low-level forwarding?
- Manipulated in assembly?

#### Risks
- Hash replay
- Malformed calldata abuse

### ✅ msg.sig
#### Functionality
- First 4 bytes of calldata (function selector).

#### Audit Focus
- Used in manual dispatch?
- Selector collision risk?

## tx (Transaction Context)
### ✅ tx.origin
#### Functionality
- Original EOA that initiated transaction.

🔥 NEVER use for auth
Classic Vulnerability
```
require(tx.origin == owner);
```

Attack:
- Malicious contract calls victim
- tx.origin remains user
- Bypass occurs

#### Audit Rule
- If you see tx.origin in auth → HIGH severity issue.

### ✅ tx.gasprice
#### Functionality
- Gas price of transaction.

#### Audit Focus
- Used in refund logic?
- Used in anti-bot mechanism?

#### Risk
- Manipulable by sender.

## block (Block Context)
### ✅ block.timestamp
#### Functionality
- Current block Unix timestamp.

#### Audit Focus
- Used in randomness?
- Used in auction?
- Used in strict comparison?

#### Risk
- Miner manipulation (~±15 seconds)
- Never use for:
    - Randomness

Critical lottery outcome

### ✅ block.number
#### Functionality
- Current block height.

#### Audit Focus
- Used for randomness?
- Used in delay logic?

### ✅ blockhash(uint blockNumber)
#### Functionality
- Returns hash of recent block (last 256 blocks).

#### Audit Focus
- Used for randomness?
- Is blockNumber validated?

#### Risk
- Predictable/manipulable in some cases.

### ✅ block.basefee
#### Functionality
- EIP-1559 base fee.

#### Audit Focus
- Used in economic logic?
- Used in gas refund calculation?

### ✅ block.chainid
#### Functionality
- Current chain ID.

#### Audit Focus
- Included in signature domain?
- Replay attack protection?

### ✅ block.coinbase
#### Functionality
- Miner/validator address.

#### Audit Focus
- Rarely used.
    - If used → potential manipulation risk.

## abi (Application Binary Interface Helpers)
### ✅ abi.encode(...)
#### Functionality
- Standard ABI encoding (safe).

#### Audit Focus
- Preferred for hashing.

### ✅ abi.encodePacked(...)
#### Dangerous
- Tightly packed encoding.

```
Collision Risk

abi.encodePacked("a", "bc")
abi.encodePacked("ab", "c")

Same hash possible.
```

#### Audit Rule
- If hashing multiple dynamic types with encodePacked → check for collision.

### ✅ abi.decode(bytes, types)
#### Functionality
- Decode encoded data.

#### Audit Focus
- Is data validated?
- Length checked?
- Assembly used instead?

### ✅ abi.encodeWithSelector(selector, ...)
#### Functionality
- Encode call with selector.

#### Audit Focus
- Is selector correct?
- Hardcoded selector verified?

### ✅ abi.encodeWithSignature("func(uint256)", ...)
#### Audit Focus
- Typo risk in string
- Signature mismatch
- Gas inefficiency

## address Type Members
### ✅ address.balance
#### Functionality
- ETH balance of address.

#### Audit Focus
- Used in logic?
- Assumes non-zero?
- Used for randomness?

### ✅ address.code
- Returns bytecode at address.

#### Audit Focus
- Used to detect contract:
```
if(addr.code.length > 0)
```
- Not reliable during constructor.

### ✅ address.codehash
- Returns keccak256 of code.

#### Audit Focus
- Used in contract validation?
- Proxy pattern?

### ✅ address.call(...)
- Low-level call.

#### Audit Focus
- Return value checked?
- Reentrancy?
- Gas forwarding issue?

### ✅ address.delegatecall(...)
🔥 MOST DANGEROUS
#### Audit Focus
- Storage collision?
- Target trusted?
- Upgrade control protected?

### ✅ address.staticcall(...)
- Read-only external call.

#### Audit Focus
- Is state modification assumed impossible?
- Is return checked?

### ✅ address.send()
- 2300 gas.
- Returns bool.

#### Audit Focus
- Return checked?
- DoS risk?

### ✅ address.transfer()
- 2300 gas.
- Reverts on failure.

⚠️ Post EIP-1884 gas cost change risk.

## type() Global
### ✅ type(uint256).max
- Max value.

#### Audit Focus
- Used in boundary checks.

### ✅ type(Contract).creationCode
#### Audit Focus
- Used in CREATE2 deployments.
- Ensure salt and bytecode validated.

### ✅ gasleft()
#### Functionality
- Returns remaining gas.

#### Audit Focus
- Used in anti-bot?
- Used in loop break?
- Can attacker manipulate?

## assert, require, revert
### ✅ assert(condition)
- For invariants only.
#### Audit Focus
- Used for input validation? (Wrong)
- Critical invariant properly defined?

### ✅ require(condition, reason)
- User validation.

#### Audit Focus
- All external inputs validated?
- Missing checks?

### ✅ revert CustomError()
- Gas efficient.

## Cryptographic Globals
### ✅ keccak256(bytes)
#### Audit Focus
- Used for randomness?
- Used for signature?
- encodePacked collision?

### ✅ ecrecover(hash, v, r, s)
- Signature recovery.

#### Audit Focus
- Nonce included?
- chainId included?
- Malleability handled?
- Signature replay protected?

## High Severity Audit Red Flags

Immediately flag if seen:
- tx.origin
- delegatecall to untrusted address
- abi.encodePacked with dynamic types for hashing
- block.timestamp for randomness
- Unchecked low-level call return
- this.function() internal call
- address.code.length used for anti-bot


## Most Dangerous Globals (Ranked)
- delegatecall
- tx.origin
- abi.encodePacked
- block.timestamp
- low-level call
- ecrecover
- msg.sender (in proxy logic)

## Auditor Memory Summary Table
- tx.origin	
    - Auth bypass -> Critical
- delegatecall
    - Storage takeover -> Critical
- encodePacked
    - Hash collision -> High
- timestamp
    - Miner manipulation -> Medium
- call()
    - Reentrancy -> High
- ecrecover
    - Replay -> High
- gasleft()
    - Manipulation logic -> Medium

# Summary — Language Features That Matter Most
<table> <tr> <th style="background-color:#34495e;color:white;">Feature</th> <th style="background-color:#34495e;color:white;">Why Critical</th>
<tr><td>storage vs memory</td><td>State corruption risk</td></tr>
<tr><td>delegatecall</td><td>Full storage takeover</td></tr>
<tr><td>visibility</td><td>Privilege exposure</td></tr>
<tr><td>modifiers</td><td>Access bypass</td></tr>
<tr><td>inheritance</td><td>Unexpected override</td></tr>
<tr><td>fallback</td><td>ETH loss</td></tr>
<tr><td>unchecked</td><td>Math exploit</td></tr>
<tr><td>encodePacked</td><td>Hash collision</td></tr>
<tr><td>initializer</td><td>Proxy takeover</td></tr>
<tr><td>type casting</td><td>Truncation exploit</td></tr>
	
</table>

# Golden Rule for Security Auditors
- Master these 5 at expert level:
- Storage layout
- delegatecall mechanics
- Inheritance resolution
- Data locations
- ABI encoding
- Everything else builds on these.

