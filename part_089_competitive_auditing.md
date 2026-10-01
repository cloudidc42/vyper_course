# Part 089: Competitive Auditing

## สารบัญ
1. Overview ของ Audit Competitions
2. Platforms: Code4rena, Sherlock, Cantina
3. วิธีค้นหา Bugs อย่างมีประสิทธิภาพ
4. การเขียน Report ที่มีคุณภาพสูง
5. Severity Classification
6. กลยุทธ์การแข่งขัน

---

## 1. Overview ของ Audit Competitions

Competitive auditing คือการแข่งขัน audit smart contracts เพื่อรับรางวัล โดยใครก็ได้ที่ค้นพบ vulnerability ที่ถูกต้องจะได้รางวัล

### Platforms หลัก

| Platform | Features | Prize Pool |
|----------|----------|------------|
| Code4rena | Open, tier system | $5k-$500k+ |
| Sherlock | Insurance-backed | $10k-$200k+ |
| Cantina | Invite + open | Variable |
| Immunefi | Bug bounty (ongoing) | Up to $10M |
| Hats Finance | Decentralized bounty | Variable |

---

## 2. Code4rena Strategy

```markdown
## Code4rena Workflow

### ขั้นตอนก่อนเริ่ม Contest

1. **อ่าน README อย่างละเอียด**
   - Scope ครอบคลุมอะไรบ้าง
   - Out-of-scope อะไร (ไม่ต้อง audit)
   - Known issues/assumptions
   - Specific areas of concern จาก team

2. **Setup Environment**
   ```bash
   git clone [contest-repo]
   cd contest
   npm install
   forge install
   forge test  # run existing tests
   ```

3. **สร้าง Notes File**
   ```
   contest_notes.md:
   - Business logic flow
   - Trust assumptions
   - External integrations
   - Initial observations
   ```

### ขั้นตอนระหว่าง Contest

1. **High-level pass (2 hours)**
   - อ่านทุก contract ผ่านๆ
   - ทำความเข้าใจ architecture
   - หา entry points
   - Note สิ่งน่าสนใจ

2. **Deep dive (ส่วนใหญ่ของเวลา)**
   - Focus ที่ complex logic
   - Follow external calls
   - Track state changes
   - Think adversarially

3. **Automated tools (parallel)**
   ```bash
   slither . --checklist
   echidna . --contract TestContract
   forge fuzz
   ```
```

---

## 3. การค้นหา Bugs อย่างมีระบบ

```python
# bug_hunting_framework.py
# Framework สำหรับ systematic bug hunting

BUG_HUNTING_APPROACH = {
    "STEP_1_ARCHITECTURE": {
        "questions": [
            "Contract ทำอะไร? (business logic)",
            "Who are the actors? (user, admin, keeper, etc.)",
            "What are the trust boundaries?",
            "ใช้ external protocols อะไรบ้าง?",
            "What are the invariants? (สิ่งที่ต้อง true เสมอ)",
        ],
        "time": "30 minutes"
    },
    
    "STEP_2_ATTACK_SURFACE": {
        "areas": [
            "External functions (entry points)",
            "Admin functions (privilege escalation?)",
            "Token transfers (ERC-20, ERC-721)",
            "ETH handling (receive, fallback, payable)",
            "External calls (callbacks, hooks)",
            "Price oracles",
            "Governance mechanisms",
        ],
        "time": "1 hour"
    },
    
    "STEP_3_VULNERABILITY_CLASSES": {
        "critical_to_check": [
            "Reentrancy (CEI violations)",
            "Access control (missing checks)",
            "Integer overflow/underflow",
            "Price manipulation",
            "Flash loan attacks",
            "Signature replay",
        ],
        "medium_to_check": [
            "Precision loss",
            "Rounding errors",
            "Front-running",
            "DoS (griefing)",
            "Business logic errors",
        ],
        "time": "2-4 hours"
    }
}

def scan_for_patterns(code: str):
    """Quick pattern scanning"""
    patterns = {
        "REENTRANCY_RISK": [
            "raw_call",
            ".transfer(",
            ".send(",
            "IERC20.transfer",
        ],
        "ORACLE_RISK": [
            "getReserves",
            "spot_price",
            "current_price",
        ],
        "ACCESS_CONTROL_RISK": [
            "tx.origin",
            "# no check",
            "anyone can",
        ],
        "ARITHMETIC_RISK": [
            "/ 10000 *",  # division before multiplication
            "* 10000 /",  # potential overflow first
        ]
    }
    
    findings = {}
    for risk_type, keywords in patterns.items():
        matches = []
        for i, line in enumerate(code.split('\n')):
            for keyword in keywords:
                if keyword in line:
                    matches.append((i+1, line.strip()))
        if matches:
            findings[risk_type] = matches
    
    return findings
```

---

## 4. การเขียน Bug Report ที่มีคุณภาพสูง

รายงานที่ดีคือสิ่งที่ทำให้ judge ยืนยัน severity ที่คุณอ้างได้ทันที

```markdown
## Report Template (High Quality)

### [H-01] Title: ชัดเจน สั้น บอก vulnerability type

**Summary:**
อธิบาย vulnerability ใน 2-3 ประโยค ให้เข้าใจได้ทันที

**Vulnerability Detail:**
อธิบายเชิงเทคนิคว่า bug เกิดขึ้นได้อย่างไร

ในฟังก์ชัน `withdraw()` บรรทัดที่ 45:
```vyper
@external
def withdraw(amount: uint256):
    # State ยังไม่ update
    raw_call(msg.sender, b"", value=amount)  # ← reentrancy point
    self.balances[msg.sender] -= amount      # ← too late
```

**Impact:**
Attacker สามารถ drain contract ของ ETH ทั้งหมดได้

**Proof of Concept:**
```solidity
contract Attacker {
    IVulnerable target;
    
    function attack() external payable {
        target.deposit{value: 1 ether}();
        target.withdraw(1 ether);
    }
    
    receive() external payable {
        if (address(target).balance >= 1 ether) {
            target.withdraw(1 ether);  // recursive call
        }
    }
}
```

**Recommended Mitigation:**
```vyper
@external
def withdraw(amount: uint256):
    # CEI: Effects before Interactions
    self.balances[msg.sender] -= amount  # ← update first
    raw_call(msg.sender, b"", value=amount)  # ← call last
```

**Tools Used:** Manual review, Slither
```

---

## 5. Severity Classification

```markdown
## Severity Framework

### Critical
- Direct theft of funds (all users)
- Permanent DoS of protocol
- Complete access control bypass

Example: Reentrancy ที่ drain ทุก ETH ใน contract

### High
- Theft of funds (some users, specific conditions)
- Protocol insolvency
- Severe DoS

Example: Oracle manipulation ที่ undervalue collateral

### Medium
- Partial fund loss (edge cases)
- Protocol disruption (temporary)
- Incorrect calculations affecting users

Example: Precision loss ทำให้ user ได้ tokens น้อยกว่าที่ควร

### Low
- Minor incorrect behavior
- Gas inefficiency ที่ผิดปกติ
- Best practices violations

Example: Missing zero address check บน non-critical function

### Informational/Gas
- Code quality issues
- Optimization suggestions
- Documentation corrections

## Factors ที่มีผลต่อ Severity

1. **Likelihood**: ยากแค่ไหนที่จะ exploit?
   - Easy (no prerequisites)
   - Medium (requires specific conditions)
   - Hard (requires large capital, complex setup)

2. **Impact**: เสียหายมากแค่ไหน?
   - High (loss of funds, permanent DoS)
   - Medium (temporary disruption)
   - Low (minor inconvenience)

Severity = max(Likelihood + Impact)
```

---

## 6. Common Winning Patterns ใน Contests

```markdown
## Patterns ที่มักชนะรางวัล

### 1. Business Logic Bugs
ไม่ได้มาจาก standard patterns แต่จาก protocol-specific logic

ตัวอย่าง: "ถ้า user withdraw ในวันเดียวกับที่ epoch เปลี่ยน
rewards จะถูก counted สองครั้ง"

ค้นหาด้วย: trace through entire user journey manually

### 2. Integration Bugs
เกิดจาก interaction กับ external protocols

ตัวอย่าง: "Protocol assume ERC-20 balanceOf accurate แต่
token X มี fee-on-transfer ทำให้ accounting틀ผิด"

ค้นหาด้วย: list all external integrations, read their docs

### 3. Rounding Errors
ทำให้ protocol ขาดทุนหรือ user เสียเปรียบ

ตัวอย่าง: "Due to rounding in fee calculation, attacker
สามารถ withdraw dust amounts ซ้ำๆ โดยไม่เสีย fee"

ค้นหาด้วย: find all division operations, trace rounding direction

### 4. State Machine Bugs
Protocol ไม่ได้จัดการ state transitions ถูกต้อง

ตัวอย่าง: "Auction สามารถ finalize ได้ก่อน minimum duration
ถ้า owner call finalize() และ extend() พร้อมกัน"

ค้นหาด้วย: draw state diagram, test all transitions

### 5. Inflation/Sandwich Attacks
Initial deposit vulnerability หรือ share manipulation

ตัวอย่าง: First depositor mint manipulation ใน ERC-4626 vaults

ค้นหาด้วย: check first deposit behavior
```

---

## 7. กลยุทธ์การแข่งขัน

```markdown
## Contest Strategy

### Time Management (1 week contest example)

Day 1-2: Understanding
- อ่าน documentation ทั้งหมด
- Map out architecture
- List external dependencies
- Identify high-value targets

Day 3-4: Deep Dive
- Focus on complex functions
- Trace critical paths
- Write PoC for suspected bugs
- Run automated tools

Day 5-6: Polish
- Write reports สำหรับ confirmed bugs
- Double-check severity
- Add PoC ให้ครบ
- Peer review (ถ้ามีทีม)

Day 7: Submission
- Final review ของ reports
- Submit ก่อน deadline
- ตรวจสอบไม่มี duplicate issues

### Tips สำหรับมือใหม่

1. **เริ่มจาก smaller scope**: หา contests ที่มี < 500 lines of code
2. **Focus on areas of concern**: Judge มักให้คะแนนสูงกับ issues ที่ team mention
3. **Read past contest reports**: เรียนรู้ patterns ที่ชนะจาก past contests
4. **Join Discord**: ถาม/ตอบคำถามกับ community
5. **Quality > Quantity**: 1 High > 10 Low findings

### Common Mistakes

1. ส่ง duplicate issues (ลด payout)
2. Over-scope severity (เสีย credibility)
3. Missing PoC (ทำให้ validate ยาก)
4. Vague description (judge ปฏิเสธ)
5. ไม่ submit gas optimization (easy points)
```

---

## 8. แบบฝึกหัด

1. เข้าร่วม Code4rena contest และส่ง report แรก
2. อ่าน 5 past contest reports จาก same protocol type
3. เขียน PoC สำหรับ reentrancy vulnerability
4. Practice severity classification บน 10 example bugs
5. สร้าง personal bug hunting checklist

---

*จบ Part 089: Competitive Auditing*
