# Part 001: แนะนำ Vyper และ Ethereum Blockchain

## สารบัญ
1. [Blockchain คืออะไร?](#blockchain)
2. [Ethereum คืออะไร?](#ethereum)
3. [Smart Contract คืออะไร?](#smart-contract)
4. [Vyper คืออะไร?](#vyper)
5. [ทำไมต้องใช้ Vyper?](#why-vyper)
6. [Vyper vs Solidity](#comparison)
7. [สถาปัตยกรรม EVM](#evm)
8. [Gas และ Transaction Fees](#gas)
9. [Ecosystem ของ Vyper](#ecosystem)
10. [เส้นทางการเรียนรู้](#learning-path)

---

## 1. Blockchain คืออะไร? {#blockchain}

Blockchain คือฐานข้อมูลแบบกระจายศูนย์ (Distributed Ledger) ที่เก็บข้อมูลในรูปแบบ "บล็อก" ที่เชื่อมต่อกันเป็นสาย

### คุณสมบัติหลักของ Blockchain

```
+----------+     +----------+     +----------+
| Block 1  |---->| Block 2  |---->| Block 3  |
|----------|     |----------|     |----------|
| Hash:    |     | Hash:    |     | Hash:    |
| 0x1a2b3c |     | 0x4d5e6f |     | 0x7a8b9c |
|----------|     |----------|     |----------|
| Prev:    |     | Prev:    |     | Prev:    |
| 0x000000 |     | 0x1a2b3c |     | 0x4d5e6f |
|----------|     |----------|     |----------|
| Txns:    |     | Txns:    |     | Txns:    |
| [tx1,tx2]|     | [tx3,tx4]|     | [tx5,tx6]|
+----------+     +----------+     +----------+
```

**คุณสมบัติสำคัญ:**
- **Decentralized**: ไม่มีจุดควบคุมกลาง
- **Immutable**: ข้อมูลที่บันทึกแล้วแก้ไขไม่ได้
- **Transparent**: ทุกคนสามารถตรวจสอบได้
- **Trustless**: ไม่ต้องอาศัยความไว้วางใจในบุคคลที่สาม

### Consensus Mechanism

Ethereum ใช้ **Proof of Stake (PoS)** หลังจาก The Merge ในปี 2022:

```
Validators (ผู้ตรวจสอบ)
    |
    ├── Stake ETH เป็น Collateral
    ├── Propose และ Attest Blocks
    ├── ได้รับ Rewards จากการทำงานถูกต้อง
    └── ถูก Slash หากทำผิดกฎ
```

---

## 2. Ethereum คืออะไร? {#ethereum}

Ethereum คือ Decentralized Computing Platform ที่รองรับ Smart Contracts

### ประวัติ Ethereum
- **2013**: Vitalik Buterin เผยแพร่ Ethereum Whitepaper
- **2015**: Ethereum Mainnet เปิดตัว (Frontier)
- **2016**: DAO Hack และ Ethereum Classic Fork
- **2017**: ICO Boom, ERC-20 Standard
- **2020**: DeFi Summer, ETH 2.0 Beacon Chain
- **2022**: The Merge (PoW → PoS)
- **2023**: Shanghai Upgrade (withdrawals enabled)
- **2024**: Dencun Upgrade (Proto-Danksharding)

### Ethereum Architecture

```
┌─────────────────────────────────────────┐
│              Application Layer           │
│  (DApps, DEX, Lending, NFT, Gaming...)  │
├─────────────────────────────────────────┤
│              Smart Contract Layer        │
│         (EVM Bytecode Execution)        │
├─────────────────────────────────────────┤
│              Consensus Layer            │
│          (Proof of Stake)               │
├─────────────────────────────────────────┤
│              P2P Network Layer          │
│         (Node Communication)            │
└─────────────────────────────────────────┘
```

### Ethereum Accounts

มี 2 ประเภท:

**1. Externally Owned Account (EOA)**
```
Address: 0x742d35Cc6634C0532925a3b8D4C9C3E3F2e1d70f
├── Private Key (ควบคุมโดยผู้ใช้)
├── Public Key
├── ETH Balance
└── Nonce (จำนวน transactions ที่ส่ง)
```

**2. Contract Account**
```
Address: 0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984
├── Contract Code (Bytecode)
├── Storage (State Variables)
├── ETH Balance
└── Nonce (จำนวน contracts ที่สร้าง)
```

---

## 3. Smart Contract คืออะไร? {#smart-contract}

Smart Contract คือโปรแกรมที่ทำงานบน Blockchain โดยอัตโนมัติ ตามเงื่อนไขที่กำหนดไว้

### แนวคิดหลัก

```
เปรียบเทียบ Smart Contract กับ Traditional Contract:

Traditional Contract:
ผู้ซื้อ ─── เซ็นสัญญา ───► ทนายความ ─── บังคับใช้ ───► ผู้ขาย
                              (Trusted Third Party)

Smart Contract:
ผู้ซื้อ ─── Deploy Code ───► Blockchain ─── Auto Execute ───► ผู้ขาย
                              (Trustless, Automated)
```

### ตัวอย่างการทำงาน Smart Contract

```
สถานการณ์: ซื้อขายบ้านผ่าน Smart Contract

1. ผู้ขายสร้าง Contract กำหนดราคา = 100 ETH
2. ผู้ซื้อส่ง 100 ETH ไปที่ Contract
3. ผู้ขายยืนยันโอนกรรมสิทธิ์
4. Contract อัตโนมัติ:
   - โอน ETH ให้ผู้ขาย
   - บันทึกความเป็นเจ้าของให้ผู้ซื้อ

ไม่มีคนกลาง, ไม่มีค่าธรรมเนียมนายหน้า, โปร่งใส 100%
```

### Properties ของ Smart Contract

| Property | คำอธิบาย |
|----------|----------|
| **Deterministic** | ผลลัพธ์เหมือนกันทุกครั้ง |
| **Isolated** | ทำงานใน Sandbox (EVM) |
| **Terminable** | มี Gas Limit ป้องกัน Infinite Loop |
| **Auditable** | Bytecode เปิดเผยบน Blockchain |

---

## 4. Vyper คืออะไร? {#vyper}

Vyper คือภาษาโปรแกรมระดับสูงสำหรับเขียน Smart Contracts บน Ethereum

### ประวัติ Vyper
- สร้างโดย Vitalik Buterin และทีม Ethereum
- เน้น Security, Simplicity, Auditability
- เวอร์ชันแรก: 2018
- เวอร์ชันปัจจุบัน: 0.4.x (2024-2026)

### ตัวอย่าง Hello World ใน Vyper

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

# ตัวแปร state ง่ายๆ
greeting: String[100]

@deploy
def __init__():
    self.greeting = "Hello, Vyper World!"

@view
@external
def get_greeting() -> String[100]:
    return self.greeting

@external
def set_greeting(new_greeting: String[100]):
    self.greeting = new_greeting
```

### ความแตกต่างจาก Python

Vyper มีไวยากรณ์คล้าย Python แต่มีข้อจำกัดสำคัญ:

```python
# Python ปกติ
class MyClass:
    def __init__(self):
        self.value = 0
    
    def set_value(self, v):
        self.value = v

# Vyper Smart Contract
# @version 0.4.0

value: uint256

@deploy
def __init__():
    self.value = 0

@external
def set_value(v: uint256):
    self.value = v
```

---

## 5. ทำไมต้องใช้ Vyper? {#why-vyper}

### Design Principles ของ Vyper

**1. Security First**
- ลด Attack Surface
- ไม่มี Feature ที่ซับซ้อนและอาจนำไปสู่ Bug
- Type Safety เข้มงวด

**2. Simplicity**
- ไวยากรณ์อ่านง่าย
- Codebase ขนาดเล็ก = Audit ง่ายกว่า
- หลักการ "Explicit is better than implicit"

**3. Auditability**
- Code ที่อ่านง่ายสำหรับ Security Auditor
- ไม่มี Hidden Control Flow
- ป้องกัน "Reentrancy by Default"

### Features ที่ Vyper ตั้งใจ **ไม่มี** (เพื่อความปลอดภัย)

| Feature | เหตุผลที่ไม่มี |
|---------|----------------|
| Inline Assembly | ยากต่อการ Audit, ข้ามการตรวจสอบ |
| Function Overloading | ทำให้ Code อ่านยาก, Confusing |
| Operator Overloading | ทำให้ Code ไม่ชัดเจน |
| Recursive Calls | ป้องกัน Stack Overflow |
| Infinite Loops | ป้องกัน Gas Drain Attack |
| Class Inheritance | ซับซ้อน, ยาก Audit |
| Multiple Inheritance | Diamond Problem |
| Modifiers (like Solidity) | ทำให้ Control Flow ไม่ชัด |

---

## 6. Vyper vs Solidity {#comparison}

### เปรียบเทียบโดยตรง

```
                    Vyper           Solidity
                    -----           --------
ไวยากรณ์:          Python-like     C++/JavaScript-like
ระดับความปลอดภัย:   สูงมาก          ปานกลาง-สูง
ความยืดหยุ่น:      น้อยกว่า         มากกว่า
ความนิยม:          น้อยกว่า         สูงมาก
Gas Efficiency:     ใกล้เคียงกัน    ใกล้เคียงกัน
ชุมชน:             เล็กกว่า         ใหญ่กว่า
Audit Cost:         ต่ำกว่า          สูงกว่า
Learning Curve:     ง่ายกว่า        ยากกว่า
```

### ตัวอย่างโค้ดเปรียบเทียบ: Simple Storage

**Solidity:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedData;
    address public owner;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    function set(uint256 x) public onlyOwner {
        storedData = x;
    }
    
    function get() public view returns (uint256) {
        return storedData;
    }
}
```

**Vyper:**
```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

storedData: uint256
owner: address

@deploy
def __init__():
    self.owner = msg.sender

@external
def set(x: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.storedData = x

@view
@external
def get() -> uint256:
    return self.storedData
```

### ความแตกต่างด้านความปลอดภัย

**1. Reentrancy Protection**
```python
# Vyper: ป้องกัน Reentrancy โดย Default
# State อัปเดตก่อนส่ง ETH เสมอ

# Solidity: ต้องใช้ ReentrancyGuard manually
# ถ้าลืม = หายนะ (เช่น DAO Hack)
```

**2. Integer Overflow**
```python
# Vyper: ตรวจสอบ Overflow/Underflow โดย Default
# ไม่ต้องใช้ SafeMath

# Solidity < 0.8.0: ต้องใช้ SafeMath
# Solidity >= 0.8.0: ตรวจสอบ Default แล้ว
```

**3. Bounds Checking**
```python
# Vyper: ตรวจสอบ Array Bounds โดย Default
arr: uint256[5]
arr[10] = 1  # Compile Error!

# Solidity: ตรวจสอบ Runtime เท่านั้น (ใน EVM)
```

---

## 7. สถาปัตยกรรม EVM {#evm}

### Ethereum Virtual Machine

EVM คือ Sandboxed Environment สำหรับ Execute Smart Contracts

```
Smart Contract Source Code
         │
         ▼ Compile
    EVM Bytecode
         │
         ▼ Deploy
    Ethereum Network
         │
         ▼ Transaction Trigger
    EVM Execution
         │
         ▼
    State Change (Storage Update)
```

### EVM Stack Machine

```
EVM เป็น Stack-based Machine:

PUSH1 0x05    →  Stack: [5]
PUSH1 0x03    →  Stack: [3, 5]
ADD           →  Stack: [8]
PUSH1 0x00    →  Stack: [0, 8]
SSTORE        →  Store 8 at slot 0
```

### EVM Storage Types

```
Memory Layout:
┌─────────────────────────────────────────┐
│  Stack (256-bit words, max 1024 items)  │
│  ► ใช้สำหรับ computation               │
├─────────────────────────────────────────┤
│  Memory (Byte array, volatile)          │
│  ► ใช้ใน Function Call เดียว           │
├─────────────────────────────────────────┤
│  Storage (Key-Value, 256-bit→256-bit)   │
│  ► Persistent, แพงที่สุด               │
├─────────────────────────────────────────┤
│  Calldata (Read-only, Input Data)       │
│  ► ถูกที่สุด, ใช้สำหรับ Function Args  │
└─────────────────────────────────────────┘
```

### Vyper → EVM Compilation

```
vyper contract.vy
         │
         ▼
┌─────────────────────┐
│   Vyper Compiler    │
│  ┌───────────────┐  │
│  │   Parser      │  │
│  └───────┬───────┘  │
│          ▼          │
│  ┌───────────────┐  │
│  │   Type Check  │  │
│  └───────┬───────┘  │
│          ▼          │
│  ┌───────────────┐  │
│  │  Code Gen     │  │
│  └───────┬───────┘  │
│          ▼          │
│   EVM Bytecode      │
└─────────────────────┘
         │
         ▼
   Deploy to Ethereum
```

---

## 8. Gas และ Transaction Fees {#gas}

### Gas คืออะไร?

Gas คือหน่วยวัด Computational Work บน Ethereum

```
Transaction Fee = Gas Used × Gas Price

Gas Price วัดใน: Gwei (1 Gwei = 10^-9 ETH)
Gas Used: จำนวน Gas ที่ใช้จริง
Gas Limit: Gas สูงสุดที่ยอมให้ใช้
```

### EIP-1559: Fee Market Reform (2021)

```
Base Fee (เผาทิ้ง) + Priority Fee (ให้ Validator)
= Effective Gas Price

Base Fee: คำนวณอัตโนมัติตาม Network Demand
Priority Fee: "Tip" ให้ Validator เพื่อเพิ่มลำดับความสำคัญ
```

### Gas Cost ของ Operations ทั่วไป

| Operation | Gas Cost | หมายเหตุ |
|-----------|----------|---------|
| SSTORE (new) | 20,000 | เขียน Storage ครั้งแรก |
| SSTORE (update) | 2,900 | แก้ไข Storage |
| SLOAD | 2,100 | อ่าน Storage |
| CALL | 700+ | เรียก Contract อื่น |
| ADD, MUL | 3 | คำนวณพื้นฐาน |
| KECCAK256 | 30+ | Hash Function |
| LOG (Event) | 375+ | บันทึก Event |

### วิธีประหยัด Gas

```python
# @version 0.4.0

# ❌ แพง: อ่าน Storage หลายครั้ง
@external
def expensive_function():
    for i: uint256 in range(10):
        self.total += self.value  # Storage read + write ทุก iteration

# ✅ ถูก: ใช้ Memory Variable
@external
def cheap_function():
    temp: uint256 = self.value  # อ่าน Storage ครั้งเดียว
    local_total: uint256 = 0
    for i: uint256 in range(10):
        local_total += temp  # Memory operation
    self.total = local_total  # เขียน Storage ครั้งเดียว
```

---

## 9. Ecosystem ของ Vyper {#ecosystem}

### Tools และ Frameworks

```
Development Tools:
├── Vyper Compiler (vyper)
├── Titanoboa (Testing Framework)
├── Brownie (Python Framework)
├── Hardhat + vyper plugin
├── Foundry + vyper support
└── Ape Framework

IDEs & Plugins:
├── VS Code + Vyper Extension
├── Remix IDE (รองรับ Vyper)
└── JetBrains + Vyper Plugin

Testing:
├── Pytest + Titanoboa
├── Hypothesis (Property-Based Testing)
└── Brownie Test Suite

Deployment:
├── Ethers.js
├── Web3.py
├── Brownie Deploy
└── Hardhat Deploy

Security:
├── Slither (Static Analyzer)
├── Mythril (Symbolic Execution)
└── Manticore (Dynamic Analysis)
```

### โปรเจกต์สำคัญที่ใช้ Vyper

| โปรเจกต์ | ประเภท | TVL/ความสำคัญ |
|----------|--------|--------------|
| Curve Finance | DEX/Stablecoin | $2B+ TVL |
| Yearn Finance | Yield Optimizer | $500M+ TVL |
| Uniswap V1 | DEX | ประวัติศาสตร์ |
| Lido (บางส่วน) | Liquid Staking | $20B+ |
| Balancer V2 (บางส่วน) | DEX | $1B+ |
| Velodrome | DEX (Optimism) | $200M+ |

---

## 10. เส้นทางการเรียนรู้ {#learning-path}

### Roadmap สำหรับผู้เริ่มต้น

```
Week 1-2: พื้นฐาน
├── Ethereum Concepts
├── Vyper Syntax
├── Basic Types & Variables
└── Simple Storage Contract

Week 3-4: ฟังก์ชันและ Control Flow
├── Functions & Visibility
├── If/Elif/Else
├── For Loops
└── Basic Testing

Week 5-6: Data Structures
├── Arrays & DynArray
├── HashMap
├── Structs
└── Events

Month 2: Token Standards
├── ERC-20 Token
├── ERC-721 NFT
└── Token Testing

Month 3-4: DeFi Patterns
├── Access Control
├── Staking
├── AMM Basics
└── Security Patterns

Month 5-6: Advanced
├── DeFi Protocol Development
├── Gas Optimization
├── Security Auditing
└── Production Deployment
```

### Resources แนะนำ

**Documentation:**
- https://docs.vyperlang.org/ (Official Docs)
- https://ethereum.org/developers/ (Ethereum Dev Portal)
- https://eips.ethereum.org/ (Ethereum Improvement Proposals)

**Learning:**
- Vyper GitHub: https://github.com/vyperlang/vyper
- Titanoboa: https://github.com/vyperlang/titanoboa
- Curve Finance Code: https://github.com/curvefi

---

## สรุป Part 001

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Blockchain และ Ethereum พื้นฐาน
- ✅ Smart Contract คืออะไรและทำงานอย่างไร
- ✅ Vyper คืออะไร และหลักการออกแบบ
- ✅ ความแตกต่างระหว่าง Vyper และ Solidity
- ✅ EVM Architecture
- ✅ Gas และ Transaction Fees
- ✅ Ecosystem และ Tools

## แบบฝึกหัด

1. ค้นคว้า: อ่าน Ethereum Whitepaper (ย่อ) ที่ ethereum.org
2. สำรวจ: เปิด Etherscan.io ดู Transaction จริงๆ
3. วิเคราะห์: หา Smart Contract ของ Curve Finance บน Etherscan แล้วสังเกตว่าเขียนด้วย Vyper
4. เปรียบเทียบ: หา Contract ที่เขียนด้วย Solidity มาเปรียบเทียบ Code Structure

---

**ต่อไป: [Part 002 - ติดตั้งและตั้งค่า Development Environment](part_002_setup.md)**
