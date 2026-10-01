# Part 088: Smart Contract Code Review

## สารบัญ
1. Code Review Methodology
2. Vyper-specific Checklist
3. Common Vulnerability Categories
4. Red Flags และ Anti-patterns
5. Review Tools
6. ตัวอย่าง Code Review จริง

---

## 1. Code Review Methodology

การทำ code review สำหรับ smart contract ต้องมีระบบ methodical เพราะผิดพลาดครั้งเดียวอาจสูญเสียเงินได้

### ขั้นตอน Code Review

```
1. CONTEXT GATHERING (30 min)
   - อ่าน documentation/whitepaper
   - เข้าใจ business logic
   - ระบุ trust boundaries
   - ทำ threat modeling เบื้องต้น

2. MANUAL REVIEW (2-4 hours)
   - อ่าน code flow ทั้งหมด
   - ตาม state changes
   - ตาม external calls
   - ตาม access control

3. AUTOMATED ANALYSIS (1-2 hours)
   - Slither/Mythril
   - Echidna fuzzing
   - Static analysis tools

4. INTEGRATION REVIEW (1 hour)
   - Dependencies และ imports
   - Protocol interactions
   - Composability risks

5. DOCUMENTATION REVIEW (30 min)
   - NatSpec correctness
   - Test coverage
   - Invariant documentation
```

---

## 2. Vyper-specific Checklist

```python
# vyper_review_checklist.py

VYPER_CHECKLIST = {
    "SYNTAX_AND_VERSION": [
        "ใช้ version pragma ชัดเจน (# @version 0.4.0)",
        "@deploy decorator ก่อน __init__",
        "DynArray[Type, N] แทน dynamic arrays",
        "from vyper.interfaces import ERC20 ที่ถูกต้อง",
        "ไม่มี fallback function ที่ไม่ตั้งใจ",
    ],
    
    "ACCESS_CONTROL": [
        "owner/admin ถูก initialize ใน __init__",
        "sensitive functions มี access modifiers",
        "modifier logic ถูกต้อง (assert ไม่ใช่ if)",
        "ไม่มี function ที่เรียกได้จาก anyone",
        "two-step ownership transfer?",
    ],
    
    "REENTRANCY": [
        "CEI pattern ใช้ตลอด",
        "@nonreentrant บน functions ที่มี external calls",
        "read-only reentrancy considered",
        "cross-function reentrancy checked",
        "ETH transfer ทำ last",
    ],
    
    "ARITHMETIC": [
        "overflow/underflow: Vyper 0.4.0 checks automatically",
        "division by zero checks",
        "precision loss ใน division",
        "rounding direction (floor vs ceil)",
        "multiplication ก่อน division",
    ],
    
    "ORACLE_AND_PRICE": [
        "ไม่ใช้ spot price สำหรับ critical calculations",
        "TWAP ใช้งาน?",
        "oracle manipulation resistance",
        "price bounds/sanity checks",
        "multiple oracle sources?",
    ],
    
    "EXTERNAL_CALLS": [
        "return values checked",
        "gas limits บน external calls",
        "trusted vs untrusted contracts",
        "call/delegatecall/staticcall ใช้ถูกต้อง",
        "ERC-777 hooks considered",
    ],
    
    "STORAGE": [
        "state variables initialized correctly",
        "no uninitialized storage",
        "proxy storage collision?",
        "storage layout ไม่ conflict",
        "immutable variables ใช้ถูกต้อง",
    ],
    
    "EVENTS": [
        "events emit บน state changes ทั้งหมด",
        "indexed parameters เหมาะสม",
        "events sufficient สำหรับ off-chain monitoring",
        "ไม่มี sensitive data ใน events",
    ],
    
    "TESTS": [
        "unit tests coverage > 95%",
        "integration tests ครอบคลุม flows",
        "invariant tests มี?",
        "edge cases ถูก test",
        "negative tests (should fail) มี?",
    ],
}

def generate_checklist_report(contract_name: str):
    print(f"=== Code Review Checklist: {contract_name} ===\n")
    for category, items in VYPER_CHECKLIST.items():
        print(f"\n[{category}]")
        for item in items:
            print(f"  □ {item}")

generate_checklist_report("MyVyperContract")
```

---

## 3. ตัวอย่าง Contract ที่มีปัญหา (สำหรับฝึก Review)

```vyper
# @version 0.4.0
# @title Buggy Token Sale
# @notice DO NOT USE - มี bugs หลายอย่าง

from vyper.interfaces import ERC20

# BUG 1: ไม่มี event สำหรับ purchase
# BUG 2: ไม่มี reentrancy protection
# BUG 3: oracle ใช้ spot price
# BUG 4: access control ไม่ครบ
# BUG 5: arithmetic อาจเกิด precision loss

owner: public(address)
token: public(address)
price: public(uint256)  # USD price per token, 18 decimals
totalRaised: public(uint256)
softCap: public(uint256)
hardCap: public(uint256)
isFinalized: public(bool)
contributions: HashMap[address, uint256]

# BUG: ไม่มี paused state

@deploy
def __init__(_token: address, _price: uint256, _softCap: uint256, _hardCap: uint256):
    self.owner = msg.sender
    self.token = _token
    self.price = _price
    self.softCap = _softCap
    self.hardCap = _hardCap

@payable
@external
def buyTokens():
    """BUG: หลายอย่าง"""
    # BUG 1: ไม่ check isFinalized
    # BUG 2: ไม่ check hardCap
    
    ethAmount: uint256 = msg.value
    
    # BUG 3: ใช้ spot ETH/USD price จาก... ที่ไหน?
    # ในนี้ assume price คือ token/ETH ratio
    tokenAmount: uint256 = ethAmount * 10**18 / self.price  # BUG: division before multiplication
    
    # BUG 4: ไม่ check token balance ของ contract
    ERC20(self.token).transfer(msg.sender, tokenAmount)
    
    self.contributions[msg.sender] += ethAmount
    self.totalRaised += ethAmount
    
    # BUG 5: ไม่ emit event

@external
def finalize():
    # BUG 6: ไม่ check ว่า caller คือ owner
    # BUG 7: ไม่ check softCap reached
    
    assert not self.isFinalized
    self.isFinalized = True
    
    # BUG 8: ส่ง ETH ไปที่ไหน? ไม่ได้ระบุ
    send(self.owner, self.balance)

@external
def refund():
    """Refund ถ้า softCap ไม่ถึง"""
    # BUG 9: ไม่ check ว่า softCap ไม่ถึงก่อน
    # BUG 10: reentrancy vulnerability!
    
    amount: uint256 = self.contributions[msg.sender]
    assert amount > 0, "No contribution"
    
    # BUG: external call ก่อน state update = reentrancy!
    raw_call(msg.sender, b"", value=amount, revert_on_failure=True)
    
    self.contributions[msg.sender] = 0  # Too late!
    self.totalRaised -= amount
```

---

## 4. Code Review Report สำหรับ Contract ข้างบน

```markdown
# Security Review Report: BuggyTokenSale.vy

## Critical Issues (ต้องแก้ก่อน deploy)

### C-01: Reentrancy in refund()
**Severity:** Critical
**Location:** refund(), lines 57-62

**Description:**
refund() ทำ external call ก่อน update state ทำให้ attacker สามารถ
reenter และดึงเงินได้หลายครั้ง

**Exploit:**
```
attacker.receive():
    if target.contributions(attacker) > 0:
        target.refund()  # recursive call!
```

**Fix:**
```vyper
@external
def refund():
    amount: uint256 = self.contributions[msg.sender]
    assert amount > 0, "No contribution"
    
    # Effects first
    self.contributions[msg.sender] = 0  # ← update state first!
    self.totalRaised -= amount
    
    # Interaction last
    raw_call(msg.sender, b"", value=amount, revert_on_failure=True)
```

### C-02: Missing access control on finalize()
**Severity:** Critical
**Location:** finalize(), line 43

**Description:**
ใครก็ได้สามารถเรียก finalize() ได้ ทำให้ sale จบก่อนกำหนด

**Fix:**
```vyper
@external
def finalize():
    assert msg.sender == self.owner, "Not owner"
    # ...
```

## High Issues

### H-01: Missing hardCap check in buyTokens()
**Severity:** High

**Description:**
ไม่มีการ check hardCap ทำให้ raise เกิน target ได้

**Fix:**
```vyper
assert self.totalRaised + msg.value <= self.hardCap, "Hard cap reached"
```

### H-02: Missing softCap check in refund()
**Severity:** High

**Description:**
ใครก็ได้ขอ refund ได้แม้ว่า softCap จะถึงแล้ว

## Medium Issues

### M-01: Precision loss in token calculation
**Severity:** Medium

**Description:**
```vyper
tokenAmount: uint256 = ethAmount * 10**18 / self.price  # potential precision loss
```
Division ก่อน multiplication ทำให้ precision loss

**Fix:**
```vyper
tokenAmount: uint256 = (ethAmount * 10**18) / self.price
```

## Low Issues

### L-01: Missing events
### L-02: No emergency pause mechanism
### L-03: Missing NatSpec documentation
```

---

## 5. Red Flags และ Anti-patterns

```vyper
# @version 0.4.0
# @title Red Flags Examples

# ===== RED FLAG 1: Dangerous raw_call without checks =====
@external
def dangerousCall(target: address, data: Bytes[1024]):
    # ⚠️ RED FLAG: arbitrary external call จาก user input
    raw_call(target, data)  # user controls target AND data!

# ===== RED FLAG 2: Block timestamp dependency =====
@view
@external
def isRandomWinner(user: address) -> bool:
    # ⚠️ RED FLAG: miners สามารถ manipulate block.timestamp
    return convert(keccak256(convert(block.timestamp, bytes32)), uint256) % 2 == 0

# ===== RED FLAG 3: tx.origin authentication =====
@external
def withdraw():
    # ⚠️ RED FLAG: tx.origin แทน msg.sender = phishing vulnerability
    assert tx.origin == self.owner, "Not owner"
    send(self.owner, self.balance)

# ===== RED FLAG 4: Unbounded loop =====
users: DynArray[address, 10000]

@external
def distributeRewards():
    # ⚠️ RED FLAG: loop ไม่มี bound → could hit gas limit
    for user: address in self.users:
        send(user, 1000)  # unbounded!

# ===== RED FLAG 5: Hardcoded addresses =====
UNISWAP_ROUTER: constant(address) = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D
# ⚠️ RED FLAG: hardcoded mainnet address ไม่ work บน testnet/L2

# ===== RED FLAG 6: Inconsistent state after failed call =====
@external
def buyAndSend(amount: uint256, recipient: address):
    # ⚠️ RED FLAG: ถ้า send ล้มเหลว state อาจ inconsistent
    self.totalSupply += amount
    ERC20(self.token).transfer(recipient, amount)  # could fail!
    # totalSupply เพิ่มแล้ว แต่ transfer ล้มเหลว

# ===== RED FLAG 7: Missing zero address check =====
owner: address

@deploy
def __init__(_owner: address):
    # ⚠️ RED FLAG: ถ้า _owner เป็น zero address → contract locked forever
    self.owner = _owner  # no zero address check!

# ===== RED FLAG 8: Integer division truncation =====
@view
@external  
def getFee(amount: uint256, rate: uint256) -> uint256:
    # ⚠️ RED FLAG: division truncation ทำให้ fee เป็น 0 สำหรับ small amounts
    return amount / 10000 * rate  # division before multiplication!
    # ควรเป็น: return amount * rate / 10000
```

---

## 6. Automated Review Tools

```bash
#!/bin/bash
# run_security_tools.sh
# Script สำหรับ run automated security analysis

CONTRACT_PATH=$1
echo "=== Running Security Analysis on: $CONTRACT_PATH ==="

# 1. Slither (Solidity/Vyper static analysis)
echo "\n--- Slither Analysis ---"
slither $CONTRACT_PATH \
    --detect reentrancy-eth,reentrancy-no-eth,unchecked-transfer \
    --json slither_report.json

# 2. Mythril (symbolic execution)
echo "\n--- Mythril Analysis ---"
myth analyze $CONTRACT_PATH \
    --execution-timeout 90 \
    -o json > mythril_report.json

# 3. Check for common patterns
echo "\n--- Pattern Check ---"
grep -n "raw_call" $CONTRACT_PATH && echo "⚠️ Found raw_call - review carefully"
grep -n "tx.origin" $CONTRACT_PATH && echo "⚠️ Found tx.origin usage"
grep -n "block.timestamp" $CONTRACT_PATH && echo "⚠️ Found block.timestamp dependency"

echo "\n=== Analysis Complete ==="
```

```python
# vyper_analyzer.py
# Python tool สำหรับ static analysis ของ Vyper contracts

import re
from pathlib import Path
from dataclasses import dataclass, field
from typing import List

@dataclass
class Finding:
    severity: str
    title: str
    description: str
    line: int
    recommendation: str

class VyperAnalyzer:
    def __init__(self, filepath: str):
        self.filepath = filepath
        self.code = Path(filepath).read_text()
        self.lines = self.code.split('\n')
        self.findings: List[Finding] = []

    def analyze(self):
        self._check_reentrancy()
        self._check_access_control()
        self._check_arithmetic()
        self._check_events()
        self._check_oracle_usage()
        return self.findings

    def _check_reentrancy(self):
        """ตรวจสอบ reentrancy patterns"""
        in_function = False
        has_external_call = False
        has_state_update_after_call = False
        has_nonreentrant = False

        for i, line in enumerate(self.lines):
            if '@nonreentrant' in line:
                has_nonreentrant = True
            if '@external' in line or '@internal' in line:
                in_function = True
                has_external_call = False
                has_state_update_after_call = False
                has_nonreentrant = False

            if in_function:
                # Check for external calls
                if 'raw_call' in line or '.transfer(' in line or '.send(' in line:
                    has_external_call = True

                # Check for state updates after external calls
                if has_external_call and ('self.' in line and '=' in line and 'raw_call' not in line):
                    has_state_update_after_call = True

                if has_external_call and has_state_update_after_call and not has_nonreentrant:
                    self.findings.append(Finding(
                        severity="HIGH",
                        title="Potential Reentrancy",
                        description="State update after external call without @nonreentrant",
                        line=i + 1,
                        recommendation="Use CEI pattern or add @nonreentrant decorator"
                    ))
                    # Reset to avoid duplicate findings
                    has_state_update_after_call = False

    def _check_access_control(self):
        """ตรวจสอบ access control patterns"""
        for i, line in enumerate(self.lines):
            # Check for tx.origin usage
            if 'tx.origin' in line and 'assert' in line:
                self.findings.append(Finding(
                    severity="HIGH",
                    title="tx.origin Authentication",
                    description="Using tx.origin for authentication is vulnerable to phishing",
                    line=i + 1,
                    recommendation="Use msg.sender instead of tx.origin"
                ))

    def _check_arithmetic(self):
        """ตรวจสอบ arithmetic issues"""
        division_pattern = re.compile(r'\w+\s*/\s*\w+\s*\*')
        for i, line in enumerate(self.lines):
            if division_pattern.search(line) and 'def ' not in line:
                self.findings.append(Finding(
                    severity="MEDIUM",
                    title="Division Before Multiplication",
                    description="Potential precision loss from division before multiplication",
                    line=i + 1,
                    recommendation="Perform multiplication before division"
                ))

    def _check_events(self):
        """ตรวจสอบ event emissions"""
        # Check if state-changing functions have events
        # Simplified check
        pass

    def _check_oracle_usage(self):
        """ตรวจสอบ oracle usage"""
        for i, line in enumerate(self.lines):
            if 'getReserves' in line:
                self.findings.append(Finding(
                    severity="MEDIUM",
                    title="Potential Spot Price Usage",
                    description="getReserves() used - ensure TWAP is used for price feeds",
                    line=i + 1,
                    recommendation="Use TWAP oracle instead of spot price"
                ))

    def print_report(self):
        print(f"\n=== SECURITY ANALYSIS REPORT ===")
        print(f"File: {self.filepath}")
        print(f"Total Findings: {len(self.findings)}\n")

        severity_order = ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW', 'INFO']

        for severity in severity_order:
            findings = [f for f in self.findings if f.severity == severity]
            if findings:
                print(f"\n[{severity}] ({len(findings)} findings)")
                for f in findings:
                    print(f"  Line {f.line}: {f.title}")
                    print(f"    {f.description}")
                    print(f"    Fix: {f.recommendation}")


# ใช้งาน
if __name__ == "__main__":
    analyzer = VyperAnalyzer("my_contract.vy")
    findings = analyzer.analyze()
    analyzer.print_report()
```

---

## 7. Review Template

```markdown
# Smart Contract Security Review

**Contract:** [Contract Name]
**Reviewer:** [Name]
**Date:** [Date]
**Version:** [Commit hash]

## Executive Summary

[2-3 ประโยคสรุป overall security posture]

## Scope

Files reviewed:
- contract.vy
- tests/test_contract.py

Out of scope:
- External dependencies

## Methodology

1. Manual code review
2. Automated static analysis (Slither, Mythril)
3. Dynamic testing (Echidna fuzzing)
4. Threat modeling

## Findings Summary

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| C-01 | Critical | Reentrancy in withdraw() | Open |
| H-01 | High | Missing access control | Open |
| M-01 | Medium | Precision loss | Open |

## Detailed Findings

### C-01: [Title]
- **Severity:** Critical
- **Location:** File:Line
- **Description:** ...
- **Impact:** ...
- **Proof of Concept:** ...
- **Recommendation:** ...

## Positive Observations

[สิ่งที่ทำได้ดี]

## Conclusion

[สรุปและคำแนะนำ]
```

---

## 8. สรุป Code Review Best Practices

1. **Systematic approach**: ใช้ checklist ทุกครั้ง
2. **Multiple passes**: อ่านหลายรอบ จากมุมต่างกัน
3. **Think like attacker**: "ถ้าฉันเป็น attacker จะทำอะไรได้?"
4. **Document findings**: เขียน PoC ให้ชัดเจน
5. **Prioritize**: Critical/High ก่อน
6. **Verify fixes**: Review fix ด้วย

---

## แบบฝึกหัด

1. Review buggy contract ข้างบนและหา bugs ทั้งหมด
2. เขียน fixed version ของ buggy contract
3. สร้าง checklist เฉพาะสำหรับ DeFi protocol
4. Run automated tools บน contract ที่มีอยู่แล้ว
5. เขียน security review report ฉบับสมบูรณ์

---

*จบ Part 088: Smart Contract Code Review*
