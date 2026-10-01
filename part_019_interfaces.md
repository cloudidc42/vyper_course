# Part 019: Interface Basics

## สารบัญ
1. [Interface คืออะไร?](#what-is-interface)
2. [Interface Declaration](#interface-declaration)
3. [implements Keyword](#implements)
4. [การเรียก External Contract](#external-calls)
5. [ERC-20 Interface](#erc20-interface)
6. [ERC-721 Interface](#erc721-interface)
7. [ตัวอย่าง: Token Aggregator Contract](#token-aggregator)
8. [แบบฝึกหัด](#exercises)

---

## 1. Interface คืออะไร? {#what-is-interface}

Interface คือ Contract Definition ที่ระบุแค่ Function Signatures โดยไม่มี Implementation

```
Interface ประกอบด้วย:
  ✓ Function Signatures (ชื่อ, Parameters, Return Types)
  ✓ State Mutability (view, pure, payable, nonpayable)
  ✗ ไม่มี Function Body
  ✗ ไม่มี State Variables
  ✗ ไม่มี Events (ประกาศได้แต่ Contract ต้อง Implement)
```

**ประโยชน์ของ Interface:**
- เรียกใช้ Contract อื่นที่ Implement Interface เดียวกัน
- กำหนด Standard ที่ Contract ต้อง Follow (เช่น ERC-20)
- Type-safe External Calls
- ลด Coupling ระหว่าง Contract

---

## 2. Interface Declaration {#interface-declaration}

### Syntax พื้นฐาน

```vyper
# @version 0.4.0

# ประกาศ Interface
interface ISimpleToken:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def approve(spender: address, amount: uint256) -> bool: nonpayable
```

### Interface กับ State Mutability

```vyper
# @version 0.4.0

interface IExample:
    # view: อ่าน State แต่ไม่เขียน
    def getValue() -> uint256: view
    
    # pure: ไม่อ่านและไม่เขียน State
    def calculate(a: uint256, b: uint256) -> uint256: pure
    
    # nonpayable: เขียน State แต่ไม่รับ ETH
    def setValue(newValue: uint256): nonpayable
    
    # payable: เขียน State และรับ ETH ได้
    def deposit(): payable
```

### Interface กับ Events

```vyper
# @version 0.4.0

interface IERC20WithEvents:
    # ประกาศ Events ใน Interface
    event Transfer:
        sender: indexed(address)
        recipient: indexed(address)
        amount: uint256
    
    event Approval:
        owner: indexed(address)
        spender: indexed(address)
        amount: uint256
    
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
```

### Interface กับ Structs

```vyper
# @version 0.4.0

# ประกาศ Struct ภายนอก Interface
struct OrderInfo:
    maker: address
    taker: address
    amount: uint256
    price: uint256
    expiry: uint256

interface IOrderBook:
    def getOrder(orderId: uint256) -> OrderInfo: view
    def fillOrder(orderId: uint256, amount: uint256) -> bool: nonpayable
    def cancelOrder(orderId: uint256): nonpayable
```

---

## 3. implements Keyword {#implements}

`implements` ใช้บอก Compiler ว่า Contract นี้ต้อง Implement Interface ที่กำหนด

### การใช้ implements

```vyper
# @version 0.4.0

interface ICounter:
    def increment(): nonpayable
    def decrement(): nonpayable
    def getCount() -> uint256: view
    def reset(): nonpayable

# บอก Compiler ว่า Contract นี้ Implement ICounter
implements: ICounter

count: uint256

@deploy
def __init__():
    self.count = 0

# ต้อง Implement ทุก Function ใน Interface
@external
def increment():
    self.count += 1

@external
def decrement():
    assert self.count > 0, "Counter: cannot decrement below zero"
    self.count -= 1

@external
@view
def getCount() -> uint256:
    return self.count

@external
def reset():
    self.count = 0
```

**ถ้าไม่ Implement บาง Function:**
```
# Compiler จะ Error:
# Contract does not implement all functions in Interface
```

### Implements หลาย Interface

```vyper
# @version 0.4.0

interface IOwnable:
    def owner() -> address: view
    def transferOwnership(newOwner: address): nonpayable

interface IPausable:
    def pause(): nonpayable
    def unpause(): nonpayable
    def paused() -> bool: view

implements: IOwnable
implements: IPausable

_owner: address
_paused: bool

@deploy
def __init__():
    self._owner = msg.sender
    self._paused = False

@external
@view
def owner() -> address:
    return self._owner

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self._owner, "Not owner"
    assert newOwner != empty(address), "Zero address"
    self._owner = newOwner

@external
def pause():
    assert msg.sender == self._owner, "Not owner"
    self._paused = True

@external
def unpause():
    assert msg.sender == self._owner, "Not owner"
    self._paused = False

@external
@view
def paused() -> bool:
    return self._paused
```

---

## 4. การเรียก External Contract {#external-calls}

### เรียก View Function

```vyper
# @version 0.4.0

interface ITokenPrice:
    def getPrice(token: address) -> uint256: view
    def getPrices(tokens: DynArray[address, 10]) -> DynArray[uint256, 10]: view

priceOracle: address

@deploy
def __init__(oracle: address):
    self.priceOracle = oracle

@external
@view
def getTokenValue(token: address, amount: uint256) -> uint256:
    """คำนวณมูลค่า Token"""
    price: uint256 = ITokenPrice(self.priceOracle).getPrice(token)
    return price * amount / 10**18
```

### เรียก State-Changing Function

```vyper
# @version 0.4.0

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def allowance(owner: address, spender: address) -> uint256: view

@external
def swapTokens(
    tokenIn: address,
    tokenOut: address,
    amountIn: uint256,
    minAmountOut: uint256
) -> uint256:
    """Swap Tokens (Simplified DEX Example)"""
    # รับ Token จาก User
    success: bool = IERC20(tokenIn).transferFrom(
        msg.sender,
        self,
        amountIn
    )
    assert success, "Transfer in failed"
    
    # คำนวณ Amount Out (Simplified)
    amountOut: uint256 = self._calculateAmountOut(amountIn)
    assert amountOut >= minAmountOut, "Insufficient output amount"
    
    # ส่ง Token ให้ User
    success = IERC20(tokenOut).transfer(msg.sender, amountOut)
    assert success, "Transfer out failed"
    
    return amountOut

@internal
def _calculateAmountOut(amountIn: uint256) -> uint256:
    # Simplified calculation
    return amountIn * 98 / 100  # 2% fee
```

### เรียก Payable Function

```vyper
# @version 0.4.0

interface IWETHGateway:
    def depositETH(onBehalfOf: address): payable
    def withdrawETH(amount: uint256, to: address): nonpayable

wethGateway: address

@deploy
def __init__(gateway: address):
    self.wethGateway = gateway

@external
@payable
def depositToLending():
    """ฝาก ETH เข้า Lending Protocol ผ่าน Gateway"""
    assert msg.value > 0, "No ETH sent"
    
    IWETHGateway(self.wethGateway).depositETH(msg.sender, value=msg.value)
```

### ตรวจสอบ Return Value

```vyper
# @version 0.4.0

interface ISafeToken:
    def transfer(to: address, amount: uint256) -> bool: nonpayable

@external
def safeTransfer(token: address, to: address, amount: uint256):
    """Transfer โดยตรวจสอบ Return Value"""
    # ตรวจสอบก่อนว่าเป็น Contract
    assert token.code_size > 0, "Not a contract"
    
    # เรียก Transfer และตรวจสอบ Result
    result: bool = ISafeToken(token).transfer(to, amount)
    assert result, "Transfer failed"
```

---

## 5. ERC-20 Interface {#erc20-interface}

### ERC-20 Interface มาตรฐาน

```vyper
# @version 0.4.0

# Standard ERC-20 Interface
interface IERC20:
    # Events
    event Transfer:
        sender: indexed(address)
        recipient: indexed(address)
        amount: uint256
    
    event Approval:
        owner: indexed(address)
        spender: indexed(address)
        amount: uint256
    
    # View Functions
    def name() -> String[100]: view
    def symbol() -> String[32]: view
    def decimals() -> uint8: view
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def allowance(owner: address, spender: address) -> uint256: view
    
    # State-Changing Functions
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def transferFrom(
        sender: address,
        to: address,
        amount: uint256
    ) -> bool: nonpayable

# Extended ERC-20 Interface
interface IERC20Extended:
    def mint(to: address, amount: uint256): nonpayable
    def burn(amount: uint256): nonpayable
    def burnFrom(account: address, amount: uint256): nonpayable
```

### ใช้ ERC-20 Interface

```vyper
# @version 0.4.0

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def approve(spender: address, amount: uint256) -> bool: nonpayable

@external
def stakingDeposit(token: address, amount: uint256):
    """ฝาก Token เข้า Staking"""
    assert amount > 0, "Zero amount"
    
    # ดึง Balance ก่อน Transfer
    balanceBefore: uint256 = IERC20(token).balanceOf(self)
    
    # Transfer Token จาก User
    success: bool = IERC20(token).transferFrom(msg.sender, self, amount)
    assert success, "Transfer failed"
    
    # ตรวจสอบว่าได้รับจริง (ป้องกัน Fee-on-Transfer Token)
    balanceAfter: uint256 = IERC20(token).balanceOf(self)
    actualReceived: uint256 = balanceAfter - balanceBefore
    assert actualReceived == amount, "Fee-on-transfer not supported"
```

---

## 6. ERC-721 Interface {#erc721-interface}

### ERC-721 Interface มาตรฐาน

```vyper
# @version 0.4.0

interface IERC721:
    # Events
    event Transfer:
        sender: indexed(address)
        receiver: indexed(address)
        tokenId: indexed(uint256)
    
    event Approval:
        owner: indexed(address)
        approved: indexed(address)
        tokenId: indexed(uint256)
    
    event ApprovalForAll:
        owner: indexed(address)
        operator: indexed(address)
        approved: bool
    
    # View Functions
    def balanceOf(owner: address) -> uint256: view
    def ownerOf(tokenId: uint256) -> address: view
    def getApproved(tokenId: uint256) -> address: view
    def isApprovedForAll(owner: address, operator: address) -> bool: view
    def supportsInterface(interfaceId: bytes4) -> bool: view
    
    # State-Changing Functions
    def approve(to: address, tokenId: uint256): nonpayable
    def setApprovalForAll(operator: address, approved: bool): nonpayable
    def transferFrom(sender: address, receiver: address, tokenId: uint256): nonpayable
    def safeTransferFrom(
        sender: address,
        receiver: address,
        tokenId: uint256
    ): nonpayable

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        sender: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable
```

---

## 7. ตัวอย่าง: Token Aggregator Contract {#token-aggregator}

Contract ที่รวม Token หลายตัวและ Provide Liquidity

```vyper
# @version 0.4.0
"""
@title Token Aggregator
@notice ตัวอย่างการใช้ Interface กับ ERC-20 Token หลายตัว
@dev รองรับ Deposit, Withdrawal และการคำนวณ Portfolio Value
"""

# ==================== Interfaces ====================

interface IERC20:
    def name() -> String[100]: view
    def symbol() -> String[32]: view
    def decimals() -> uint8: view
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def allowance(owner: address, spender: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable

interface IPriceOracle:
    def getPrice(token: address) -> uint256: view  # Price in USD (18 decimals)
    def getPrices(tokens: DynArray[address, 20]) -> DynArray[uint256, 20]: view

# ==================== Events ====================

event Deposited:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event TokenAdded:
    token: indexed(address)
    name: String[100]

event TokenRemoved:
    token: indexed(address)

# ==================== Structs ====================

struct TokenInfo:
    token: address
    name: String[100]
    symbol: String[32]
    decimals: uint8
    isActive: bool
    totalDeposited: uint256

# ==================== State Variables ====================

owner: public(address)
priceOracle: public(address)

supportedTokens: public(DynArray[address, 20])
tokenInfo: HashMap[address, TokenInfo]
tokenSupported: HashMap[address, bool]

# User balances: user => token => amount
userBalances: HashMap[address, HashMap[address, uint256]]

MAX_TOKENS: constant(uint256) = 20

@deploy
def __init__(oracle: address):
    assert oracle != empty(address), "Invalid oracle"
    assert oracle.code_size > 0, "Oracle is not a contract"
    
    self.owner = msg.sender
    self.priceOracle = oracle

# ==================== Admin Functions ====================

@external
def addToken(token: address):
    """
    @notice เพิ่ม Token ที่รองรับ
    """
    assert msg.sender == self.owner, "Not owner"
    assert token != empty(address), "Invalid token"
    assert token.code_size > 0, "Not a contract"
    assert not self.tokenSupported[token], "Token already supported"
    assert len(self.supportedTokens) < MAX_TOKENS, "Max tokens reached"
    
    # ดึงข้อมูล Token
    tokenName: String[100] = IERC20(token).name()
    tokenSymbol: String[32] = IERC20(token).symbol()
    tokenDecimals: uint8 = IERC20(token).decimals()
    
    self.tokenInfo[token] = TokenInfo({
        token: token,
        name: tokenName,
        symbol: tokenSymbol,
        decimals: tokenDecimals,
        isActive: True,
        totalDeposited: 0
    })
    
    self.tokenSupported[token] = True
    self.supportedTokens.append(token)
    
    log TokenAdded(token, tokenName)

@external
def removeToken(token: address):
    """
    @notice ลบ Token ออกจากรายการ
    """
    assert msg.sender == self.owner, "Not owner"
    assert self.tokenSupported[token], "Token not supported"
    
    # ตรวจสอบว่าไม่มี User ที่ถือ Token นี้อยู่
    assert self.tokenInfo[token].totalDeposited == 0, "Token has deposits"
    
    self.tokenInfo[token].isActive = False
    self.tokenSupported[token] = False
    
    # ลบออกจาก Array
    newTokens: DynArray[address, 20] = []
    for t: address in self.supportedTokens:
        if t != token:
            newTokens.append(t)
    self.supportedTokens = newTokens
    
    log TokenRemoved(token)

@external
def updateOracle(newOracle: address):
    """
    @notice อัปเดต Price Oracle
    """
    assert msg.sender == self.owner, "Not owner"
    assert newOracle != empty(address), "Invalid oracle"
    assert newOracle.code_size > 0, "Not a contract"
    
    self.priceOracle = newOracle

# ==================== User Functions ====================

@external
def deposit(token: address, amount: uint256):
    """
    @notice ฝาก Token เข้า Aggregator
    """
    assert self.tokenSupported[token], "Token not supported"
    assert self.tokenInfo[token].isActive, "Token not active"
    assert amount > 0, "Zero amount"
    
    # ตรวจสอบ Allowance
    currentAllowance: uint256 = IERC20(token).allowance(msg.sender, self)
    assert currentAllowance >= amount, "Insufficient allowance"
    
    # ดึง Balance ก่อน Transfer (ป้องกัน Fee-on-transfer)
    balanceBefore: uint256 = IERC20(token).balanceOf(self)
    
    # Transfer Token จาก User
    success: bool = IERC20(token).transferFrom(msg.sender, self, amount)
    assert success, "Transfer failed"
    
    # คำนวณจำนวนที่รับจริง
    balanceAfter: uint256 = IERC20(token).balanceOf(self)
    actualAmount: uint256 = balanceAfter - balanceBefore
    
    # อัปเดต State
    self.userBalances[msg.sender][token] += actualAmount
    self.tokenInfo[token].totalDeposited += actualAmount
    
    log Deposited(msg.sender, token, actualAmount)

@external
def withdraw(token: address, amount: uint256):
    """
    @notice ถอน Token ออกจาก Aggregator
    """
    assert self.tokenSupported[token], "Token not supported"
    assert amount > 0, "Zero amount"
    assert self.userBalances[msg.sender][token] >= amount, "Insufficient balance"
    
    # Update State ก่อน Transfer (Checks-Effects-Interactions)
    self.userBalances[msg.sender][token] -= amount
    self.tokenInfo[token].totalDeposited -= amount
    
    # Transfer Token ให้ User
    success: bool = IERC20(token).transfer(msg.sender, amount)
    assert success, "Transfer failed"
    
    log Withdrawn(msg.sender, token, amount)

@external
def withdrawAll():
    """
    @notice ถอน Token ทั้งหมดออก
    """
    for token: address in self.supportedTokens:
        amount: uint256 = self.userBalances[msg.sender][token]
        if amount > 0:
            self.userBalances[msg.sender][token] = 0
            self.tokenInfo[token].totalDeposited -= amount
            
            success: bool = IERC20(token).transfer(msg.sender, amount)
            assert success, concat("Transfer failed for token")
            
            log Withdrawn(msg.sender, token, amount)

# ==================== View Functions ====================

@external
@view
def getUserBalance(user: address, token: address) -> uint256:
    """ดู Balance ของ User สำหรับ Token"""
    return self.userBalances[user][token]

@external
@view
def getUserPortfolio(user: address) -> (
    DynArray[address, 20],
    DynArray[uint256, 20]
):
    """
    @notice ดู Portfolio ของ User
    @return tokens, amounts
    """
    tokens: DynArray[address, 20] = []
    amounts: DynArray[uint256, 20] = []
    
    for token: address in self.supportedTokens:
        balance: uint256 = self.userBalances[user][token]
        if balance > 0:
            tokens.append(token)
            amounts.append(balance)
    
    return tokens, amounts

@external
@view
def getPortfolioValue(user: address) -> uint256:
    """
    @notice คำนวณมูลค่ารวมของ Portfolio (USD, 18 decimals)
    """
    totalValue: uint256 = 0
    
    for token: address in self.supportedTokens:
        balance: uint256 = self.userBalances[user][token]
        if balance > 0:
            price: uint256 = IPriceOracle(self.priceOracle).getPrice(token)
            decimals: uint8 = self.tokenInfo[token].decimals
            
            # Normalize to 18 decimals
            normalizedBalance: uint256 = balance
            if decimals < 18:
                normalizedBalance = balance * (10 ** convert(18 - decimals, uint256))
            elif decimals > 18:
                normalizedBalance = balance / (10 ** convert(decimals - 18, uint256))
            
            value: uint256 = normalizedBalance * price / 10**18
            totalValue += value
    
    return totalValue

@external
@view
def getSupportedTokens() -> DynArray[address, 20]:
    """ดูรายการ Token ที่รองรับ"""
    return self.supportedTokens

@external
@view
def getTokenInfo(token: address) -> TokenInfo:
    """ดูข้อมูลของ Token"""
    assert self.tokenSupported[token], "Token not supported"
    return self.tokenInfo[token]

@external
@view
def getTotalValueLocked() -> uint256:
    """
    @notice คำนวณ TVL (Total Value Locked) ของ Aggregator
    """
    totalValue: uint256 = 0
    
    for token: address in self.supportedTokens:
        info: TokenInfo = self.tokenInfo[token]
        if info.totalDeposited > 0:
            price: uint256 = IPriceOracle(self.priceOracle).getPrice(token)
            decimals: uint8 = info.decimals
            
            normalizedBalance: uint256 = info.totalDeposited
            if decimals < 18:
                normalizedBalance = info.totalDeposited * (10 ** convert(18 - decimals, uint256))
            
            value: uint256 = normalizedBalance * price / 10**18
            totalValue += value
    
    return totalValue
```

### Test Suite สำหรับ Token Aggregator

```python
# tests/test_token_aggregator.py
import pytest
import boa

# Mock Price Oracle
MOCK_ORACLE_SOURCE = """
# @version 0.4.0
prices: HashMap[address, uint256]

@deploy
def __init__():
    pass

@external
def setPrice(token: address, price: uint256):
    self.prices[token] = price

@external
@view
def getPrice(token: address) -> uint256:
    return self.prices[token]

@external
@view
def getPrices(tokens: DynArray[address, 20]) -> DynArray[uint256, 20]:
    result: DynArray[uint256, 20] = []
    for token: address in tokens:
        result.append(self.prices[token])
    return result
"""

# Mock ERC-20 Token
MOCK_TOKEN_SOURCE = """
# @version 0.4.0
name: public(String[100])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

@deploy
def __init__(_name: String[100], _symbol: String[32]):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.totalSupply = 10**24
    self.balances[msg.sender] = 10**24

@external
def transfer(to: address, amount: uint256) -> bool:
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    self.allowances[sender][msg.sender] -= amount
    self.balances[sender] -= amount
    self.balances[to] += amount
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    return True

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]
"""

@pytest.fixture
def deployer():
    return boa.env.eoa

@pytest.fixture
def oracle():
    return boa.loads(MOCK_ORACLE_SOURCE)

@pytest.fixture
def token_a(deployer):
    return boa.loads(MOCK_TOKEN_SOURCE, "Token A", "TKNA")

@pytest.fixture
def token_b(deployer):
    return boa.loads(MOCK_TOKEN_SOURCE, "Token B", "TKNB")

@pytest.fixture
def aggregator(oracle):
    return boa.load("contracts/TokenAggregator.vy", oracle.address)

def test_add_token(aggregator, token_a, oracle):
    """ทดสอบเพิ่ม Token"""
    oracle.setPrice(token_a.address, 10**18)  # $1 per token
    aggregator.addToken(token_a.address)
    
    assert token_a.address in aggregator.getSupportedTokens()
    info = aggregator.getTokenInfo(token_a.address)
    assert info[2] == "TKNA"  # symbol

def test_deposit(aggregator, token_a, oracle):
    """ทดสอบ Deposit"""
    oracle.setPrice(token_a.address, 10**18)
    aggregator.addToken(token_a.address)
    
    amount = 10**20  # 100 tokens
    token_a.approve(aggregator.address, amount)
    aggregator.deposit(token_a.address, amount)
    
    assert aggregator.getUserBalance(boa.env.eoa, token_a.address) == amount

def test_withdraw(aggregator, token_a, oracle):
    """ทดสอบ Withdraw"""
    oracle.setPrice(token_a.address, 10**18)
    aggregator.addToken(token_a.address)
    
    amount = 10**20
    token_a.approve(aggregator.address, amount)
    aggregator.deposit(token_a.address, amount)
    
    initial_balance = token_a.balanceOf(boa.env.eoa)
    aggregator.withdraw(token_a.address, amount)
    
    assert token_a.balanceOf(boa.env.eoa) == initial_balance + amount
    assert aggregator.getUserBalance(boa.env.eoa, token_a.address) == 0

def test_portfolio_value(aggregator, token_a, token_b, oracle):
    """ทดสอบคำนวณมูลค่า Portfolio"""
    # Set prices
    oracle.setPrice(token_a.address, 2 * 10**18)  # $2
    oracle.setPrice(token_b.address, 5 * 10**18)  # $5
    
    aggregator.addToken(token_a.address)
    aggregator.addToken(token_b.address)
    
    # Deposit 100 TKNA ($200) + 50 TKNB ($250) = $450
    token_a.approve(aggregator.address, 100 * 10**18)
    aggregator.deposit(token_a.address, 100 * 10**18)
    
    token_b.approve(aggregator.address, 50 * 10**18)
    aggregator.deposit(token_b.address, 50 * 10**18)
    
    value = aggregator.getPortfolioValue(boa.env.eoa)
    assert value == 450 * 10**18  # $450

def test_withdraw_all(aggregator, token_a, token_b, oracle):
    """ทดสอบถอนทั้งหมด"""
    oracle.setPrice(token_a.address, 10**18)
    oracle.setPrice(token_b.address, 10**18)
    aggregator.addToken(token_a.address)
    aggregator.addToken(token_b.address)
    
    amount_a = 10**20
    amount_b = 5 * 10**19
    
    token_a.approve(aggregator.address, amount_a)
    aggregator.deposit(token_a.address, amount_a)
    
    token_b.approve(aggregator.address, amount_b)
    aggregator.deposit(token_b.address, amount_b)
    
    aggregator.withdrawAll()
    
    assert aggregator.getUserBalance(boa.env.eoa, token_a.address) == 0
    assert aggregator.getUserBalance(boa.env.eoa, token_b.address) == 0
```

---

## 8. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Yield Optimizer
สร้าง Contract ที่ Deposit Token ไปยัง Protocol ต่างๆ เพื่อ Maximize Yield

### แบบฝึกหัดที่ 2: Flash Loan Receiver
สร้าง Contract ที่ Implement Interface สำหรับรับ Flash Loan

### แบบฝึกหัดที่ 3: NFT Marketplace
สร้าง Marketplace ที่ใช้ Interface สำหรับ ERC-721 Token

---

## สรุป

| Concept | Syntax | Use Case |
|---------|--------|----------|
| ประกาศ Interface | `interface IName:` | กำหนด Contract API |
| Implement Interface | `implements: IName` | บังคับ Contract Implement |
| เรียก External | `IName(addr).func()` | เรียก Contract อื่น |
| View Call | `.func(): view` | อ่าน State จาก Contract อื่น |
| Payable Call | `.func(value=amount)` | ส่ง ETH ไปด้วย |

---

[← Part 018: Error Handling](part_018_error_handling.md) | [Part 020: Testing →](part_020_testing.md)
