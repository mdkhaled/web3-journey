# EVM Gas Optimization Patterns & Audit Red Flags

## EVM Data Location Overview

<ul>
  <li><span style="color:#d9534f;"><b>Storage</b></span> → Persistent, highest gas cost</li>
  <li><span style="color:#f0ad4e;"><b>Memory</b></span> → Temporary, mutable</li>
  <li><span style="color:#5cb85c;"><b>Calldata</b></span> → Read-only, cheapest input</li>
  <li><span style="color:#5bc0de;"><b>Stack</b></span> → Execution-level, very cheap</li>
  <li><span style="color:#9370db;"><b>Code</b></span> → Immutable bytecode</li>
  <li><span style="color:#20c997;"><b>Logs (Events)</b></span> → Off-chain observability</li>
</ul>

---

# Gas Optimization Patterns

---

## 🔴 Storage (Very High Gas Cost)

### ✅ Recommended Patterns
- Minimize storage writes (read once, write once)
- Cache storage values in memory before modification
- Use `immutable` and `constant` instead of storage
- Pack variables to share storage slots
- Delete unused storage for gas refunds
- Prefer mappings over arrays

```solidity
uint _count = count; //Read data to memory
_count++; // DO operations in memory
count = _count; //Write data back to memory
```

### ❌ Anti-Patterns
- Storage writes inside loops
- Using storage for temporary values
- Repeated reads from the same slot
- Unbounded array or mapping growth

---

## 🟠 Memory (Medium Gas Cost)

### ✅ Recommended Patterns
- Use memory only when mutation is required
- Reuse memory variables
- Prefer fixed-size arrays where possible
- Limit dynamic memory allocation

### ❌ Anti-Patterns
- Copying calldata to memory unnecessarily
- Creating large memory arrays repeatedly
- Returning large memory objects

---

## 🟢 Calldata (Low Gas Cost)

### ✅ Recommended Patterns
- Use external functions
- Accept arrays and structs as calldata
- Avoid copying calldata unless modification is required
- function process(uint[] calldata data) external {}

### ❌ Anti-Patterns
- Using public instead of external
- Blind calldata-to-memory copying

---

## 🔵 Stack (Very Low Gas Cost)

### ✅ Recommended Patterns
- Keep expressions simple
- Split complex logic into smaller functions
- Let the compiler manage stack usage

### ❌ Anti-Patterns
- Too many local variables
- Deeply nested expressions
- Ignoring stack-depth limitations

---

## 🟣 Code (Immutable)

### ✅ Recommended Patterns
- Embed constants in bytecode
- Reuse internal functions
- Keep bytecode size minimal

### ❌ Anti-Patterns
- Large constructor logic
- Code duplication
- Hardcoded magic numbers

## 🟢 Logs / Events (Medium Gas Cost)

### ✅ Recommended Patterns
- Use events for historical data
- Index only frequently queried fields
- Emit events after state changes

### ❌ Anti-Patterns
- Over-indexing event parameters
- Emitting events inside loops
- Using events as state storage

---

# Audit Red Flags by Data Location

---

## 🔴 Storage — High Risk Area
<table> <tr style="background-color:#f8d7da;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Storage writes inside loops</td> <td>Gas-based DoS vulnerability</td> </tr> <tr> <td>Unbounded arrays or mappings</td> <td>State bloat</td> </tr> <tr> <td>Re-entrancy before state update</td> <td>Critical exploit risk</td> </tr> <tr> <td>Missing access control</td> <td>Unauthorized state modification</td> </tr> <tr> <td>Storage used for temp data</td> <td>Severe gas inefficiency</td> </tr> </table>

---

## 🟠 Memory — Medium Risk
<table> <tr style="background-color:#fff3cd;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Large memory allocations</td> <td>Gas spikes</td> </tr> <tr> <td>Excessive memory copying</td> <td>Execution inefficiency</td> </tr> <tr> <td>Returning large arrays</td> <td>Expensive external calls</td> </tr> </table>

---

## 🟢 Calldata — Low Risk
<table> <tr style="background-color:#e6ffed;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Unnecessary copying</td> <td>Gas waste</td> </tr> <tr> <td>`public` instead of `external`</td> <td>Higher execution cost</td> </tr> <tr> <td>Attempted mutation</td> <td>Compile-time failure</td> </tr> </table>

---

## 🔵 Stack — Logic Risk
<table> <tr style="background-color:#e6f4ff;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Stack too deep</td> <td>Compilation failure</td> </tr> <tr> <td>Overloaded expressions</td> <td>Hidden logic bugs</td> </tr> <tr> <td>Poor variable scoping</td> <td>Audit complexity</td> </tr> </table>

---

## 🟣 Code — Design Risk
<table> <tr style="background-color:#f2e9ff;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Large bytecode size</td> <td>Deployment failure</td> </tr> <tr> <td>No upgrade strategy</td> <td>Permanent lock-in</td> </tr> <tr> <td>Hardcoded constants</td> <td>Poor maintainability</td> </tr> </table>

---

## 🟢 Logs / Events — Observability Risk
<table> <tr style="background-color:#e6fffa;"> <th>Red Flag</th> <th>Risk</th> </tr> <tr> <td>Critical data only in events</td> <td>Not accessible on-chain</td> </tr> <tr> <td>Missing events for state changes</td> <td>Poor traceability</td> </tr> <tr> <td>Over-indexed parameters</td> <td>Gas waste</td> </tr> </table>

---

##  Auditor Golden Rules
<blockquote style="border-left:6px solid #6c757d; padding-left:12px; color:#444;"> <b>State</b> lives in storage.<br/> <b>Computation</b> lives in memory.<br/> <b>Inputs</b> belong in calldata.<br/> <b>Logic</b> lives in code.<br/> <b>History</b> belongs in events. </blockquote>





---

# Summary of data locations

<table> <thead> <tr> <th>Data Location</th> <th>Where Stored</th> <th>Lifetime</th> <th>Gas Cost</th> <th>Mutability</th> <th>Typical Use Cases</th> <th>Key Notes</th> </tr> </thead> <tbody> <tr style="background-color:#ffe6e6;"> <td><b>Storage</b></td> <td>Blockchain state</td> <td>Persistent</td> <td style="color:red;"><b>Very High</b></td> <td>Mutable</td> <td>State variables, balances, mappings</td> <td>Most expensive, permanent</td> </tr> <tr style="background-color:#fff5cc;"> <td><b>Memory</b></td> <td>EVM temporary memory</td> <td>Function execution</td> <td style="color:orange;"><b>Medium</b></td> <td>Mutable</td> <td>Temporary arrays, calculations</td> <td>Cleared after execution</td> </tr> <tr style="background-color:#e6ffe6;"> <td><b>Calldata</b></td> <td>Transaction input</td> <td>External call</td> <td style="color:green;"><b>Low</b></td> <td>Immutable</td> <td>External function parameters</td> <td>Cheapest, read-only</td> </tr> <tr style="background-color:#e6f0ff;"> <td><b>Stack</b></td> <td>EVM stack</td> <td>Instruction-level</td> <td style="color:green;"><b>Very Low</b></td> <td>Mutable</td> <td>Arithmetic, local values</td> <td>Max 1024 slots</td> </tr> <tr style="background-color:#f0e6ff;"> <td><b>Code</b></td> <td>Contract bytecode</td> <td>Permanent</td> <td style="color:green;"><b>Very Low</b></td> <td>Immutable</td> <td>Business logic</td> <td>Cannot be modified</td> </tr> <tr style="background-color:#e6ffff;"> <td><b>Logs (Events)</b></td> <td>Transaction logs</td> <td>Permanent</td> <td style="color:orange;"><b>Medium</b></td> <td>Immutable</td> <td>Off-chain indexing</td> <td>Not readable on-chain</td> </tr> </tbody> </table>