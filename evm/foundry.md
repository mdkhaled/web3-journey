# Foundry Cheat Sheet for Security Audits
What matters for:
- Vulnerability reproduction
- Exploit simulation
- Fuzzing
- Invariant testing
- Fork testingv
- Gas analysis
- Storage inspection

## Core Commands
Compile
```
forge build
```
### Audit Use:
- Verify compiler version
- Check warnings
- Confirm optimizer settings

Run Tests
```
forge test
```
With verbosity:
```
forge test -vvvv
```
### Audit Use:
- See full call traces
- Inspect reverts
- Debug exploit paths

Match Specific Test
```
forge test --match-test testReentrancy
```
Match Contract
```
forge test --match-contract ExploitTest
```
## Verbosity Levels (Critical for Audit)
- Flag → Meaning
- -v → Logs
- -vv → Execution traces
- -vvv → Traces for failing tests
- -vvvv → Full traces for all tests

For deep audit debugging:
```
forge test -vvvv
```
## Cheatcodes (Most Important Section)

Cheatcodes are accessed via:
```
vm.*
```
### 🔴 Prank / Caller Manipulation
Simulate Different msg.sender
```
vm.prank(attacker);
```

Next call executed as attacker.

Multiple Calls as Attacker
```
vm.startPrank(attacker);
// multiple calls
vm.stopPrank();
```

#### Audit Use:
- Test access control bypass
- Simulate phishing
- Test multi-call attack chain

### 🔴 Deal (Set ETH Balance)
```
vm.deal(attacker, 100 ether);
```

#### Audit Use:
- Simulate flash loan attacker
- Ensure attacker has funds

### 🔴 Warp (Manipulate Time)
```
vm.warp(block.timestamp + 1 days);
```

#### Audit Use:
- Test timelocks
- Vesting bypass
- Auction expiry

### 🔴 Roll (Change Block Number)
```
vm.roll(block.number + 100);
```

#### Audit Use:
- Test block-based logic
- Emission schedule manipulation

### 🔴  Expect Revert
```
vm.expectRevert();
```

With error:
```
vm.expectRevert("Unauthorized");
```

Custom error:
```
vm.expectRevert(Unauthorized.selector);
```

#### Audit Use:
- Confirm protections actually work

### 🔴  Expect Emit
vm.expectEmit(true, true, false, true);
emit Transfer(...);

#### Audit Use:
- Verify critical event emitted

### 🔴  Record Logs
```
vm.recordLogs();
Vm.Log[] memory logs = vm.getRecordedLogs();
```

#### Audit Use:
- Inspect event emissions manually

### 🔴  Storage Manipulation (Powerful)
Read Storage
```
bytes32 slot = vm.load(address(contract), bytes32(uint256(0)));
```
Write Storage
```
vm.store(address(contract), slot, newValue);
```
#### Audit Use:
- Simulate storage corruption
- Test upgrade collisions
- Modify balances artificially

⚠️ Extremely powerful for attack simulation.

### 🔴  Snapshot / Revert State
```
uint snap = vm.snapshot();
vm.revertTo(snap);
```
#### Audit Use:
- Re-run exploit paths quickly

### 🔴  Expect Call
```
vm.expectCall(address(target), abi.encodeWithSelector(...));
```
#### Audit Use:
- Ensure external calls happen
- Validate oracle interaction

### 🔴  Mock Call
```
vm.mockCall(
    address(oracle),
    abi.encodeWithSelector(oracle.getPrice.selector),
    abi.encode(1000)
);
```

#### Audit Use:
- Simulate oracle manipulation
- Fake return values

### 🔴  Fuzzing Inputs

Test function:
```
function testFuzz_Deposit(uint256 amount) public {
```
Foundry automatically fuzzes input.

Restrict range:
```
vm.assume(amount > 0 && amount < 1000 ether);
```

#### Audit Use:
- Discover overflow
- Detect edge cases
- Break invariant assumptions

### 🔴 Invariant Testing (VERY IMPORTANT)

Create contract:
```
contract InvariantTest is Test {
```

Use:
```
function invariant_totalSupply() public {
```

Run:
```
forge test --match-contract InvariantTest
```

#### Audit Use:
- Check protocol invariants
- Detect broken accounting
- Simulate random sequences
- This is how professional audits detect deep logic flaws.

## Fork Testing (Real Mainnet State)
Create Fork
```
vm.createSelectFork("mainnet", blockNumber);
```

Or CLI:
```
forge test --fork-url $RPC_URL
```

### Audit Use:
- Test against real liquidity pools
- Simulate flash loan attacks
- Interact with real deployed contracts

## Gas Analysis
- Gas Snapshot
```
forge snapshot
```

### Audit Use:
- Detect gas griefing risk
- Check expensive loops

## Coverage
```
forge coverage
```

### Audit Use:
- Detect untested logic
- Identify attack surface

## Trace Debugging
```
forge test -vvvv
```
Shows:
- Call stack
- Internal calls
- Storage reads/writes
- Gas consumption
- Professional auditors live in this mode.

## Inspect Storage Layout
```
forge inspect Contract storageLayout
```

### Audit Use:
- Check upgrade safety
- Validate slot order
- Detect collision risk

## Inspect Bytecode
forge inspect Contract bytecode


Runtime:
```
forge inspect Contract deployedBytecode
```

### Audit Use:
- Verify optimizer
- Check metadata
- Compare deployed vs compiled

## Script Simulation
```
forge script script/Exploit.s.sol --fork-url $RPC_URL
```

### Audit Use:
- Reproduce real-world exploit
- Simulate attack transactions

# Advanced Audit Patterns Using Foundry
Simulate Reentrancy
- Create attacker contract
- Use vm.prank
- Call vulnerable function
- Inspect trace
- Simulate Flash Loan
- Fork mainnet
- Impersonate whale
- Borrow tokens
- Execute attack
- Storage Slot Attack Simulation
    - vm.store(address(proxy), slot, maliciousValue);
- Test proxy takeover scenario.

# Most Powerful Cheatcodes for Auditors (Ranked)
- vm.prank / startPrank
- vm.store
- vm.mockCall
- vm.createSelectFork
- Invariant testing
- vm.expectRevert
- vm.assume
- vm.warp
- vm.roll
- vm.load

# Professional Auditor Workflow in Foundry
- Write minimal exploit test
- Fuzz vulnerable function
- Write invariant test
- Fork mainnet
- Simulate economic attack
- Inspect storage layout
- Compare optimized vs non-optimized

# Red Flags Found During Foundry Audit
- Invariant fails after fuzzing
- Unexpected storage mutation
- External call not reverted
- Event not emitted
- Delegatecall writing wrong slot
- Time warp breaks vesting logic