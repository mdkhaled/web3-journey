# Storage Layout Orientation (How EVM Packs Variables)
## Slot Packing Rules
- Each storage slot = 32 bytes
- Smaller variables are packed into same slot
- Order matters
- Structs follow packing rules
- Mappings & dynamic arrays use hashing

✅ Efficient Packing Example
```
contract Packed {
    uint128 a;  // 16 bytes
    uint128 b;  // 16 bytes → same slot
}
```

✔ Uses 1 slot (32 bytes)

❌ Inefficient Packing Example
```
contract NotPacked {
    uint128 a;  // 16 bytes
    uint256 b;  // 32 bytes → new slot
    uint128 c;  // 16 bytes → new slot
}

Uses 3 slots instead of 2
```

## Orientation by Variable Type

<table> <tr> <th style="background-color:#34495e;color:white;">Type</th> <th style="background-color:#34495e;color:white;">Stored In</th><th style="background-color:#34495e;color:white;">Notes</th>
<tr><td>uint / bool / address</td><td>Storage slot</td><td>Packed if possible</td></tr>
<tr><td>mapping</td><td>Hash-based slot</td><td>Not iterable</td></tr>
<tr><td>dynamic array</td><td>Length in slot + data via keccak</td><td>Expensive</td></tr>
<tr><td>struct</td><td>Sequential packing</td><td>Depends on order</td></tr>
<tr><td>string / bytes</td><td>Dynamic storage</td><td>Expensive</td></tr>
</table>

## Audit Red Flags
### 🔴 Storage Issues
- Writing to storage inside loops
- Unpacked variables
- Using storage instead of memory for temp data
- Unbounded dynamic arrays

# Accessibility – Cheat Sheet
## Variable Accessibility (Visibility)
<table> <tr> <th style="background-color:#34495e;color:white;">Visibility</th> <th style="background-color:#34495e;color:white;">Accessible Inside</th> <th style="background-color:#34495e;color:white;">Accessible Outside</th> <th style="background-color:#34495e;color:white;">Notes</th> </tr> <tr> <td style="background-color:#fdecea;">🔴 private</td> <td>Same contract</td> <td>❌ No</td> <td>Still <b>visible on blockchain storage</b></td> </tr> <tr> <td style="background-color:#fff3cd;">🟡 internal</td> <td>Contract + Inherited</td> <td>❌ No</td> <td>Default for state variables</td> </tr> <tr> <td style="background-color:#d4edda;">🟢 public</td> <td>Everywhere</td> <td>✅ Yes</td> <td>Auto getter generated</td> </tr> <tr> <td style="background-color:#cce5ff;">🔵 external</td> <td>Only external calls</td> <td>✅ Yes</td> <td>Cheaper for calldata params</td> </tr> </table>

## Audit Red Flags

### 🟡 Visibility Issues
- Sensitive data marked public
- Missing private for internal logic
- Misused external vs public
- Relying on private for secrecy