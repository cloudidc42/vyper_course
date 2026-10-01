# Part 021: ERC-20 Token Standard

## สารบัญ
1. [ERC-20 คืออะไร?](#what-is-erc20)
2. [ERC-20 Specification](#specification)
3. [Required Functions](#required-functions)
4. [Events](#events)
5. [Allowance Mechanism](#allowance)
6. [Transfer Flows](#transfer-flows)
7. [Security Considerations](#security)
8. [แบบฝึกหัด](#exercises)

---

## 1. ERC-20 คืออะไร? {#what-is-erc20}

ERC-20 (Ethereum Request for Comments 20) คือมาตรฐานสำหรับ Fungible Token บน Ethereum

### ประวัติ

```
พ.ศ. 2558 (2015): Fabian Vogelsteller เสนอ EIP-20
พ.ศ. 2559 (2016): Accepted as Standard
ปัจจุบัน: Token นับพันๆ ล้าน USD ใช้ ERC-20
```

### ทำไม ERC-20 ถึงสำคัญ?

**ก่อน ERC-20:**
```
Token A: function sendTokens(address, uint)
Token B: function moveBalance(address, uint256)
Token C: function xfer(address, amount)

→ ไม่มีมาตรฐาน ทุก App ต้องเขียน Code แยกสำหรับทุก Token
```

**หลัง ERC-20:**
```
ทุก Token: function transfer(address to, uint256 amount) -> bool

→ มาตรฐานเดียว ทุก DEX/Wallet/DApp ทำงานกับทุก Token ได้
```

### Fungible vs Non-Fungible

```
Fungible (ERC-20):
  1 ETH = 1 ETH (แลกกันได้)
  1 USDT = 1 USDT (แลกกันได้)
  
Non-Fungible (ERC-721):
  NFT #1 ≠ NFT #2 (แต่ละชิ้นไม่เหมือนกัน)
```

---

## 2. ERC-20 Specification {#specification}

ERC-20 กำหนด:
1. **Functions** - 6 Functions ที่ต้อง Implement
2. **Events** - 2 Events ที่ต้อง Emit
3. **Optional** - name, symbol, decimals (แนะนำให้มี)

### Interface ERC-20 ฉบับสมบูรณ์

```vyper
# @version 0.4.0
# ERC-20 Interface (Informational)

interface IERC20:
    # ==================== Optional (แนะนำ) ====================
    def name() -> String[100]: view
    def symbol() -> String[32]: view
    def decimals() -> uint8: view
    
    # ==================== Required ====================
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def allowance(owner: address, spender: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def transferFrom(
        from_: address,
        to: address,
        amount: uint256
    ) -> bool: nonpayable
    
    # ==================== Required Events ====================
    event Transfer:
        from_: indexed(address)
        to: indexed(address)
        value: uint256
    
    event Approval:
        owner: indexed(address)
        spender: indexed(address)
        value: uint256
```

---

## 3. Required Functions {#required-functions}

### 3.1 totalSupply()

```
function totalSupply() external view returns (uint256)
```

คืน Supply ทั้งหมดของ Token ที่อยู่ใน Circulation

```vyper
# @version 0.4.0

totalSupply: public(uint256)

@deploy
def __init__(initialSupply: uint256):
    self.totalSupply = initialSupply
    
# totalSupply() ถูก Generate อัตโนมัติจาก public()
```

**ข้อพิจารณา:**
- ต้องอัปเดตเมื่อ Mint หรือ Burn
- ไม่รวม Token ที่ถูก Burn ไปยัง Dead Address (ขึ้นอยู่กับ Implementation)
- บางโปรเจคมี `maxSupply` แยกต่างหาก

### 3.2 balanceOf(account)

```
function balanceOf(address account) external view returns (uint256)
```

คืน Token Balance ของ Address ที่ระบุ

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]
```

**ข้อพิจารณา:**
- ต้องไม่ Revert สำหรับ Address ใดๆ (รวมถึง Zero Address)
- Zero Address ควรคืน 0

### 3.3 transfer(to, amount)

```
function transfer(address to, uint256 amount) external returns (bool)
```

โอน Token จาก `msg.sender` ไปยัง `to`

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]

@external
def transfer(to: address, amount: uint256) -> bool:
    """
    @notice โอน Token ไปยัง address อื่น
    @dev ต้อง Emit Transfer Event
    @return True ถ้าสำเร็จ (Revert ถ้าล้มเหลว)
    """
    assert to != empty(address), "ERC20: transfer to zero address"
    assert self.balances[msg.sender] >= amount, "ERC20: insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256
```

**ข้อพิจารณา:**
- ต้อง Emit `Transfer` Event
- ต้อง Revert ถ้าล้มเหลว (ไม่ใช่คืน False)
- Transfer 0 amount ต้องสำเร็จ (ตาม EIP)
- Transfer ไป Zero Address ควร Revert

### 3.4 approve(spender, amount)

```
function approve(address spender, uint256 amount) external returns (bool)
```

อนุญาตให้ `spender` ใช้ Token ของ `msg.sender` ในจำนวน `amount`

```vyper
# @version 0.4.0

allowances: HashMap[address, HashMap[address, uint256]]

@external
def approve(spender: address, amount: uint256) -> bool:
    """
    @notice อนุญาต spender ให้ใช้ token ของ msg.sender
    @dev ต้อง Emit Approval Event
    @return True ถ้าสำเร็จ
    """
    assert spender != empty(address), "ERC20: approve to zero address"
    
    self.allowances[msg.sender][spender] = amount
    
    log Approval(msg.sender, spender, amount)
    return True

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256
```

**ข้อพิจารณา:**
- การ `approve(spender, newAmount)` **แทนที่** Amount เดิม (ไม่บวก)
- ต้อง Emit `Approval` Event
- Allowance เป็น 0 หมายความว่า ไม่มีสิทธิ์

### 3.5 allowance(owner, spender)

```
function allowance(address owner, address spender) external view returns (uint256)
```

คืน Allowance ที่ `owner` อนุญาตให้ `spender`

```vyper
# @version 0.4.0

allowances: HashMap[address, HashMap[address, uint256]]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]
```

### 3.6 transferFrom(from, to, amount)

```
function transferFrom(address from, address to, uint256 amount) external returns (bool)
```

โอน Token จาก `from` ไปยัง `to` โดยใช้ Allowance

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    """
    @notice โอน Token แทนผู้อื่น (ต้องมี Allowance)
    @dev Reduce Allowance ยกเว้น Infinite Approval
    """
    assert sender != empty(address), "ERC20: transfer from zero address"
    assert to != empty(address), "ERC20: transfer to zero address"
    assert self.balances[sender] >= amount, "ERC20: insufficient balance"
    
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    
    if currentAllowance != max_value(uint256):  # ไม่ใช่ Infinite Approval
        assert currentAllowance >= amount, "ERC20: insufficient allowance"
        self.allowances[sender][msg.sender] = currentAllowance - amount
        log Approval(sender, msg.sender, currentAllowance - amount)
    
    self.balances[sender] -= amount
    self.balances[to] += amount
    
    log Transfer(sender, to, amount)
    return True
```

---

## 4. Events {#events}

### 4.1 Transfer Event

```
event Transfer(address indexed from, address indexed to, uint256 value)
```

ต้อง Emit เมื่อ:
1. Token ถูกโอน (`transfer` หรือ `transferFrom`)
2. Token ถูก Mint (`from` = Zero Address)
3. Token ถูก Burn (`to` = Zero Address)

```vyper
# @version 0.4.0

event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

balances: HashMap[address, uint256]
totalSupply: uint256

@deploy
def __init__(initialSupply: uint256):
    self.totalSupply = initialSupply
    self.balances[msg.sender] = initialSupply
    
    # Emit Transfer สำหรับ Initial Mint
    log Transfer(empty(address), msg.sender, initialSupply)

@external
def burn(amount: uint256):
    assert self.balances[msg.sender] >= amount
    
    self.balances[msg.sender] -= amount
    self.totalSupply -= amount
    
    # Emit Transfer สำหรับ Burn
    log Transfer(msg.sender, empty(address), amount)
```

### 4.2 Approval Event

```
event Approval(address indexed owner, address indexed spender, uint256 value)
```

ต้อง Emit เมื่อ `approve` ถูกเรียก หรือ Allowance เปลี่ยนแปลง

---

## 5. Allowance Mechanism {#allowance}

### 5.1 ทำไมต้องมี Allowance?

```
Scenario: Alice ต้องการให้ DEX ขาย Token ของเธอโดยอัตโนมัติ

ไม่มี Allowance:
  Alice ต้องเรียก DEX.swap() ด้วยตัวเอง
  → ไม่รองรับ Order Book, Automated Trading

มี Allowance:
  Alice: token.approve(DEX, 1000)  # อนุญาต DEX ใช้ 1000 Token
  DEX: token.transferFrom(Alice, Bob, 1000)  # DEX ขาย Token แทน Alice
  → Automated Trading สำเร็จ!
```

### 5.2 Allowance Flow

```
1. Alice → token.approve(DEX_address, 1000)
   allowances[Alice][DEX] = 1000

2. DEX → token.transferFrom(Alice, Bob, 500)
   allowances[Alice][DEX] = 500  (ลดลง)
   balances[Alice] -= 500
   balances[Bob] += 500

3. DEX → token.transferFrom(Alice, Charlie, 600)
   ❌ Revert! allowance(500) < amount(600)
```

### 5.3 Approval Race Condition

**ปัญหา:**
```
1. Alice approve(Mallory, 100)  # allowances[Alice][Mallory] = 100
2. Alice ต้องการเปลี่ยนเป็น 200
3. Alice approve(Mallory, 200)  # ส่ง Transaction
4. Mallory เห็น Transaction ก่อน Execute
5. Mallory: transferFrom(Alice, ..., 100)  # ใช้ allowance 100 ก่อน
6. Alice's approve(200) Execute
7. Mallory: transferFrom(Alice, ..., 200)  # ใช้ allowance 200 อีกครั้ง!
   → Mallory ได้ 300 Token แทนที่จะได้ 200
```

**วิธีแก้:**
```vyper
# @version 0.4.0

allowances: HashMap[address, HashMap[address, uint256]]

@external
def approve(spender: address, amount: uint256) -> bool:
    # วิธีที่ 1: Set เป็น 0 ก่อน แล้วค่อย Set ใหม่
    # แต่ใน 1 Transaction ไม่ปลอดภัยเท่ากัน
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# วิธีที่ 2 (ปลอดภัยกว่า): increaseAllowance / decreaseAllowance
@external
def increaseAllowance(spender: address, addedValue: uint256) -> bool:
    """เพิ่ม Allowance อย่างปลอดภัย"""
    newAllowance: uint256 = self.allowances[msg.sender][spender] + addedValue
    self.allowances[msg.sender][spender] = newAllowance
    log Approval(msg.sender, spender, newAllowance)
    return True

@external
def decreaseAllowance(spender: address, subtractedValue: uint256) -> bool:
    """ลด Allowance อย่างปลอดภัย"""
    currentAllowance: uint256 = self.allowances[msg.sender][spender]
    assert currentAllowance >= subtractedValue, "Decreased below zero"
    
    newAllowance: uint256 = currentAllowance - subtractedValue
    self.allowances[msg.sender][spender] = newAllowance
    log Approval(msg.sender, spender, newAllowance)
    return True

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256
```

### 5.4 Infinite Approval (max_value)

```vyper
# @version 0.4.0

# Infinite Approval Pattern
# ผู้ใช้ Approve max_value(uint256) หมายความว่า "ไม่จำกัด"

allowances: HashMap[address, HashMap[address, uint256]]

@external
def approveInfinite(spender: address) -> bool:
    """Infinite Approval"""
    self.allowances[msg.sender][spender] = max_value(uint256)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    
    # ถ้าเป็น Infinite Approval ไม่ต้องลด Allowance
    if currentAllowance != max_value(uint256):
        assert currentAllowance >= amount, "Insufficient allowance"
        self.allowances[sender][msg.sender] -= amount
    
    # โอน Token...
    return True
```

---

## 6. Transfer Flows {#transfer-flows}

### 6.1 Direct Transfer

```
Alice → token.transfer(Bob, 100) → Bob

Flow:
1. ตรวจสอบ balances[Alice] >= 100
2. balances[Alice] -= 100
3. balances[Bob] += 100
4. Emit Transfer(Alice, Bob, 100)
```

### 6.2 Delegated Transfer (via Allowance)

```
Alice → token.approve(DEX, 1000)
DEX → token.transferFrom(Alice, Bob, 100)

Flow:
1. ตรวจสอบ balances[Alice] >= 100
2. ตรวจสอบ allowances[Alice][DEX] >= 100
3. balances[Alice] -= 100
4. balances[Bob] += 100
5. allowances[Alice][DEX] -= 100
6. Emit Approval(Alice, DEX, 900)  // Optional แต่แนะนำ
7. Emit Transfer(Alice, Bob, 100)
```

### 6.3 Mint Flow

```
Minter → token.mint(Alice, 1000)

Flow:
1. ตรวจสอบ Minter Permission
2. totalSupply += 1000
3. balances[Alice] += 1000
4. Emit Transfer(0x0, Alice, 1000)  // ← จาก Zero Address
```

### 6.4 Burn Flow

```
Alice → token.burn(500)

Flow:
1. ตรวจสอบ balances[Alice] >= 500
2. balances[Alice] -= 500
3. totalSupply -= 500
4. Emit Transfer(Alice, 0x0, 500)  // ← ไปยัง Zero Address
```

### 6.5 Permit Flow (EIP-2612)

```
Alice Sign: { spender: DEX, value: 1000, nonce: 0, deadline: ... }
DEX: token.permit(Alice, DEX, 1000, deadline, v, r, s)

ผลเหมือน Alice เรียก approve(DEX, 1000)
แต่ไม่ต้องส่ง Transaction แยก!
```

---

## 7. Security Considerations {#security}

### 7.1 Integer Overflow/Underflow

```vyper
# Vyper 0.4.0 มี Built-in Overflow Protection
# แต่ยังต้องระวัง Logic Errors

# ✅ ปลอดภัย: Vyper จะ Revert ถ้า Overflow
self.balances[to] += amount

# ❌ อันตราย: ถ้า Logic ผิด
# ถ้า balances[sender] < amount → Underflow → Revert (ดี)
# แต่ถ้า totalSupply ไม่ถูก Update → State Inconsistent (แย่)
```

### 7.2 Zero Address Checks

```vyper
# @version 0.4.0

@external
def transfer(to: address, amount: uint256) -> bool:
    # ✅ ตรวจสอบ Zero Address
    assert to != empty(address), "Transfer to zero address"
    
    # ❌ ไม่มีการตรวจสอบ → Token หายไป!
    # self.balances[empty(address)] += amount
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    return True

balances: HashMap[address, uint256]
```

### 7.3 Reentrancy

```
ERC-20 Transfer ปกติไม่เสี่ยง Reentrancy
แต่ต้องระวังถ้า:
  - Contract รับ ETH ด้วย
  - มีการเรียก External Contract
  - ใช้ Hooks (ERC-777 มีปัญหานี้)
```

```vyper
# @version 0.4.0

# ✅ Pattern ที่ปลอดภัย: อัปเดต State ก่อนเรียก External
@external
def burnAndRefund(amount: uint256):
    # อัปเดต State ก่อน
    self.balances[msg.sender] -= amount
    self.totalSupply -= amount
    
    # แล้วค่อย External Call
    log Transfer(msg.sender, empty(address), amount)
    send(msg.sender, amount * self.ethPerToken)

balances: HashMap[address, uint256]
totalSupply: uint256
ethPerToken: uint256
```

### 7.4 Front-Running

```
Approval Race Condition ที่กล่าวถึงข้างต้น
ใช้ increaseAllowance/decreaseAllowance แทน approve
หรือใช้ Permit (EIP-2612) สำหรับ Gasless Approval
```

### 7.5 Fee-on-Transfer Tokens

```
บาง Token หัก Fee ตอน Transfer:
  transfer(Bob, 100) → Bob ได้ 98 Token (Fee 2%)

DEX ต้องจัดการ:
  balanceBefore = token.balanceOf(this)
  token.transferFrom(Alice, this, 100)
  balanceAfter = token.balanceOf(this)
  actualReceived = balanceAfter - balanceBefore  // อาจ < 100
```

### 7.6 Decimals

```
ERC-20 decimals: uint8

ค่าทั่วไป:
  18 decimals: ETH, WETH, ส่วนใหญ่ของ Token
  6 decimals: USDC, USDT
  8 decimals: WBTC

1 ETH = 1 * 10^18 Wei = 1,000,000,000,000,000,000

การคำนวณ:
  amount = 100 * 10^decimals  // 100 tokens in smallest unit
```

---

## 8. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: ERC-20 Audit
ตรวจสอบ ERC-20 Contract ที่กำหนดให้ หา Bug และ Security Issues

### แบบฝึกหัดที่ 2: Token Interaction
เขียน Contract ที่ Interact กับ ERC-20 Token:
- รับ Token เป็น Payment
- คืน Token หลัง Service

### แบบฝึกหัดที่ 3: Allowance Management
สร้าง Safe Allowance Manager ที่:
- ป้องกัน Race Condition
- มี Expiry Time
- Revoke ได้ทันที

---

## สรุป ERC-20 Standard

```
ERC-20 Functions:
  Required:
    totalSupply() → uint256
    balanceOf(address) → uint256
    transfer(address, uint256) → bool
    approve(address, uint256) → bool
    allowance(address, address) → uint256
    transferFrom(address, address, uint256) → bool
    
  Optional (แนะนำ):
    name() → string
    symbol() → string
    decimals() → uint8

ERC-20 Events:
  Required:
    Transfer(indexed address, indexed address, uint256)
    Approval(indexed address, indexed address, uint256)
```

---

[← Part 020: Testing](part_020_testing.md) | [Part 022: ERC-20 Implementation →](part_022_erc20_impl.md)
