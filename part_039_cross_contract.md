# Part 039: Cross-Contract Calls

## สารบัญ
1. [Interface-based Calls](#interface-based-calls)
2. [raw_call()](#raw_call)
3. [staticcall equivalent](#staticcall-equivalent)
4. [Return Value Handling](#return-value-handling)
5. [ตัวอย่าง: DeFi Aggregator](#ตัวอย่าง-defi-aggregator)
6. [Test Code](#test-code)

---

## Interface-based Calls

### การกำหนด Interface ใน Vyper

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Cross-Contract Calls - Interface Pattern

# กำหนด Interface สำหรับ ERC20 Token
interface ERC20:
    def name() -> String[100]: view
    def symbol() -> String[32]: view
    def decimals() -> uint8: view
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def allowance(owner: address, spender: address) -> uint256: view

# Interface สำหรับ ERC721 NFT
interface ERC721:
    def ownerOf(tokenId: uint256) -> address: view
    def safeTransferFrom(from_: address, to: address, tokenId: uint256): nonpayable
    def transferFrom(from_: address, to: address, tokenId: uint256): nonpayable
    def approve(to: address, tokenId: uint256): nonpayable
    def isApprovedForAll(owner: address, operator: address) -> bool: view
    def setApprovalForAll(operator: address, approved: bool): nonpayable

# Interface สำหรับ Uniswap V2 Pair
interface IUniswapV2Pair:
    def token0() -> address: view
    def token1() -> address: view
    def getReserves() -> (uint112, uint112, uint32): view
    def swap(
        amount0Out: uint256,
        amount1Out: uint256,
        to: address,
        data: Bytes[256]
    ): nonpayable

# ==================== ตัวอย่างการใช้งาน ====================

@view
@external
def get_token_info(token: address) -> (String[100], String[32], uint8, uint256):
    """ดึงข้อมูล token จาก interface"""
    t: ERC20 = ERC20(token)
    return t.name(), t.symbol(), t.decimals(), t.totalSupply()

@external
def transfer_token(token: address, to: address, amount: uint256):
    """โอน token ผ่าน interface"""
    ERC20(token).transfer(to, amount)

@external
def batch_transfer(
    token: address,
    recipients: DynArray[address, 100],
    amounts: DynArray[uint256, 100]
):
    """โอน token ให้หลายคนพร้อมกัน"""
    assert len(recipients) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(recipients):
            break
        ERC20(token).transferFrom(msg.sender, recipients[i], amounts[i])
```

### Interface Inheritance Pattern

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token Wrapper ใช้ Interface

interface IERC20:
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable

interface IWETH:
    def deposit(): payable
    def withdraw(amount: uint256): nonpayable
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable

WETH_ADDRESS: constant(address) = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2

owner: public(address)

@deploy
def __init__():
    self.owner = msg.sender

@external
@payable
def wrap_eth():
    """แปลง ETH เป็น WETH"""
    assert msg.value > 0, "Send ETH"
    IWETH(WETH_ADDRESS).deposit(value=msg.value)

@external
def unwrap_weth(amount: uint256):
    """แปลง WETH กลับเป็น ETH"""
    IWETH(WETH_ADDRESS).withdraw(amount)
    send(msg.sender, amount)

@view
@external
def get_weth_balance() -> uint256:
    return IWETH(WETH_ADDRESS).balanceOf(self)

@external
@payable
def __default__():
    """รับ ETH จาก WETH.withdraw()"""
    pass
```

---

## raw_call()

### raw_call คืออะไร?

`raw_call()` ใช้เมื่อไม่มี Interface หรือต้องการ flexibility มากขึ้น

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title raw_call Examples

@external
def call_with_raw(
    target: address,
    data: Bytes[1000]
) -> Bytes[1000]:
    """เรียก contract ด้วย raw data"""
    response: Bytes[1000] = b""
    success: bool = False
    
    response = raw_call(
        target,
        data,
        max_outsize=1000,
        is_static_call=False
    )
    
    return response

@external
def raw_transfer(token: address, to: address, amount: uint256) -> bool:
    """
    Transfer ERC20 โดยใช้ raw_call
    สร้าง calldata ด้วยตัวเอง
    """
    # ERC20 transfer(address,uint256) selector = 0xa9059cbb
    # Encode calldata
    calldata: Bytes[68] = concat(
        b"\xa9\x05\x9c\xbb",  # transfer selector
        convert(to, bytes32),  # to address (padded to 32 bytes)
        convert(amount, bytes32)  # amount
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32
    )
    
    if len(response) > 0:
        return convert(response, bool)
    return True

@external
@payable
def call_with_value(target: address, data: Bytes[256]):
    """ส่ง ETH พร้อมกับ call"""
    raw_call(
        target,
        data,
        value=msg.value,
        max_outsize=0
    )

@external
def delegatecall_example(target: address, data: Bytes[256]) -> Bytes[256]:
    """
    Delegatecall - รัน code ของ target ในบริบทของ contract นี้
    ใช้สำหรับ Proxy pattern
    """
    response: Bytes[256] = raw_call(
        target,
        data,
        max_outsize=256,
        is_delegate_call=True
    )
    return response
```

### Safe raw_call Pattern

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Safe raw_call Pattern

@internal
def _safe_transfer(token: address, to: address, amount: uint256):
    """
    safeTransfer - handle ทั้ง ERC20 ที่ return bool และไม่ return
    """
    if token == empty(address):
        # ส่ง ETH แทน
        send(to, amount)
        return
    
    calldata: Bytes[68] = concat(
        b"\xa9\x05\x9c\xbb",
        convert(to, bytes32),
        convert(amount, bytes32)
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32
    )
    
    if len(response) > 0:
        assert convert(response, bool), "Transfer failed"

@internal
def _safe_transfer_from(
    token: address,
    from_: address,
    to: address,
    amount: uint256
):
    """safeTransferFrom"""
    calldata: Bytes[100] = concat(
        b"\x23\xb8\x72\xdd",  # transferFrom selector
        convert(from_, bytes32),
        convert(to, bytes32),
        convert(amount, bytes32)
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32
    )
    
    if len(response) > 0:
        assert convert(response, bool), "TransferFrom failed"

@internal
def _safe_approve(token: address, spender: address, amount: uint256):
    """safeApprove"""
    calldata: Bytes[68] = concat(
        b"\x09\x5e\xa7\xb3",  # approve selector
        convert(spender, bytes32),
        convert(amount, bytes32)
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32
    )
    
    if len(response) > 0:
        assert convert(response, bool), "Approve failed"
```

---

## staticcall equivalent

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Staticcall Examples

interface IPriceFeed:
    def getPrice(token: address) -> uint256: view

@view
@external
def read_only_call(target: address, data: Bytes[256]) -> Bytes[256]:
    """
    Static call - read-only, ไม่เปลี่ยน state
    """
    response: Bytes[256] = raw_call(
        target,
        data,
        max_outsize=256,
        is_static_call=True
    )
    return response

@view
@external
def get_balance_via_static(token: address, account: address) -> uint256:
    """ดึง balance ด้วย static call"""
    calldata: Bytes[36] = concat(
        b"\x70\xa0\x82\x31",  # balanceOf selector
        convert(account, bytes32)
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32,
        is_static_call=True
    )
    
    return convert(response, uint256)
```

---

## Return Value Handling

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Return Value Handling

interface ILendingPool:
    def borrow(
        asset: address,
        amount: uint256,
        interestRateMode: uint256,
        referralCode: uint16,
        onBehalfOf: address
    ): nonpayable
    
    def getUserAccountData(user: address) -> (
        uint256,  # totalCollateralETH
        uint256,  # totalDebtETH
        uint256,  # availableBorrowsETH
        uint256,  # currentLiquidationThreshold
        uint256,  # ltv
        uint256   # healthFactor
    ): view

interface IFlashLoanReceiver:
    def executeOperation(
        assets: DynArray[address, 10],
        amounts: DynArray[uint256, 10],
        premiums: DynArray[uint256, 10],
        initiator: address,
        params: Bytes[256]
    ) -> bool: nonpayable

lending_pool: public(address)
owner: public(address)

@deploy
def __init__(_lending_pool: address):
    self.owner = msg.sender
    self.lending_pool = _lending_pool

@view
@external
def get_user_health(user: address) -> uint256:
    """ดึง health factor ของ user"""
    total_collateral: uint256 = 0
    total_debt: uint256 = 0
    available_borrows: uint256 = 0
    liquidation_threshold: uint256 = 0
    ltv: uint256 = 0
    health_factor: uint256 = 0
    
    (total_collateral, total_debt, available_borrows,
     liquidation_threshold, ltv, health_factor) = \
        ILendingPool(self.lending_pool).getUserAccountData(user)
    
    return health_factor

@view
@external
def is_position_safe(user: address) -> bool:
    """ตรวจสอบว่า position ปลอดภัย"""
    health_factor: uint256 = self.get_user_health(user)
    return health_factor >= 10**18  # HF >= 1.0
```

---

## ตัวอย่าง: DeFi Aggregator

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title DeFi Aggregator
# @notice Contract ที่รวม DEX หลายๆ ตัวเพื่อหา best price

interface IERC20:
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable

interface IUniswapV2Router:
    def getAmountsOut(
        amountIn: uint256,
        path: DynArray[address, 5]
    ) -> DynArray[uint256, 5]: view
    
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable
    
    def swapExactETHForTokens(
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: payable
    
    def swapExactTokensForETH(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable

interface IUniswapV3Router:
    def exactInputSingle(params: ExactInputSingleParams) -> uint256: payable

struct ExactInputSingleParams:
    tokenIn: address
    tokenOut: address
    fee: uint24
    recipient: address
    deadline: uint256
    amountIn: uint256
    amountOutMinimum: uint256
    sqrtPriceLimitX96: uint160

# ==================== Constants ====================

MAX_ROUTERS: constant(uint256) = 10
MAX_SLIPPAGE_BPS: constant(uint256) = 500  # 5%
PROTOCOL_FEE_BPS: constant(uint256) = 10   # 0.1%

# ==================== Events ====================

event Swapped:
    user: indexed(address)
    token_in: indexed(address)
    token_out: indexed(address)
    amount_in: uint256
    amount_out: uint256
    router_used: address

event RouterAdded:
    router: indexed(address)
    name: String[50]

event RouterRemoved:
    router: indexed(address)

# ==================== Structs ====================

struct RouterInfo:
    router_address: address
    name: String[50]
    router_type: uint8  # 0=V2, 1=V3
    active: bool

struct SwapQuote:
    router: address
    amount_out: uint256
    gas_estimate: uint256

# ==================== State Variables ====================

owner: public(address)
fee_recipient: public(address)
routers: public(DynArray[RouterInfo, MAX_ROUTERS])
router_count: public(uint256)
weth: public(address)

is_paused: public(bool)
collected_fees: public(HashMap[address, uint256])

# ==================== Constructor ====================

@deploy
def __init__(_weth: address, _fee_recipient: address):
    self.owner = msg.sender
    self.weth = _weth
    self.fee_recipient = _fee_recipient

# ==================== Admin Functions ====================

@external
def add_router(
    router_address: address,
    name: String[50],
    router_type: uint8
):
    """เพิ่ม router ใหม่"""
    assert msg.sender == self.owner, "Not owner"
    assert router_address != empty(address), "Invalid router"
    assert self.router_count < MAX_ROUTERS, "Max routers"
    
    self.routers.append(RouterInfo({
        router_address: router_address,
        name: name,
        router_type: router_type,
        active: True
    }))
    self.router_count += 1
    
    log RouterAdded(router_address, name)

@external
def remove_router(index: uint256):
    """ปิดใช้งาน router"""
    assert msg.sender == self.owner, "Not owner"
    assert index < self.router_count, "Invalid index"
    
    router_addr: address = self.routers[index].router_address
    self.routers[index].active = False
    
    log RouterRemoved(router_addr)

# ==================== Quote Functions ====================

@view
@internal
def _get_v2_quote(
    router: address,
    amount_in: uint256,
    path: DynArray[address, 5]
) -> uint256:
    """ดึง quote จาก Uniswap V2-style router"""
    amounts: DynArray[uint256, 5] = IUniswapV2Router(router).getAmountsOut(
        amount_in,
        path
    )
    
    if len(amounts) < 2:
        return 0
    
    return amounts[len(amounts) - 1]

@view
@external
def get_best_quote(
    token_in: address,
    token_out: address,
    amount_in: uint256
) -> (address, uint256):
    """
    @notice หา router ที่ให้ราคาดีที่สุด
    @return (best_router, best_amount_out)
    """
    best_router: address = empty(address)
    best_amount: uint256 = 0
    
    path: DynArray[address, 5] = [token_in, token_out]
    
    for router_info: RouterInfo in self.routers:
        if not router_info.active:
            continue
        
        if router_info.router_type == 0:  # V2
            quote: uint256 = self._get_v2_quote(
                router_info.router_address,
                amount_in,
                path
            )
            
            if quote > best_amount:
                best_amount = quote
                best_router = router_info.router_address
    
    return best_router, best_amount

@view
@external
def get_all_quotes(
    token_in: address,
    token_out: address,
    amount_in: uint256
) -> DynArray[SwapQuote, MAX_ROUTERS]:
    """ดึง quote จากทุก router"""
    quotes: DynArray[SwapQuote, MAX_ROUTERS] = []
    path: DynArray[address, 5] = [token_in, token_out]
    
    for router_info: RouterInfo in self.routers:
        if not router_info.active:
            continue
        
        quote: uint256 = 0
        if router_info.router_type == 0:
            quote = self._get_v2_quote(router_info.router_address, amount_in, path)
        
        quotes.append(SwapQuote({
            router: router_info.router_address,
            amount_out: quote,
            gas_estimate: 150000  # estimate
        }))
    
    return quotes

# ==================== Swap Functions ====================

@external
@nonreentrant
def swap_tokens(
    token_in: address,
    token_out: address,
    amount_in: uint256,
    min_amount_out: uint256,
    use_router: address  # 0x0 = auto select best
) -> uint256:
    """
    @notice Swap tokens โดย auto-route ไปยัง best DEX
    """
    assert not self.is_paused, "Paused"
    assert amount_in > 0, "Must swap > 0"
    
    # รับ tokens จาก user
    IERC20(token_in).transferFrom(msg.sender, self, amount_in)
    
    # หัก protocol fee
    fee: uint256 = amount_in * PROTOCOL_FEE_BPS / 10000
    amount_after_fee: uint256 = amount_in - fee
    self.collected_fees[token_in] += fee
    
    # หา best router ถ้าไม่ระบุ
    router: address = use_router
    if router == empty(address):
        best_router: address = empty(address)
        best_amount: uint256 = 0
        (best_router, best_amount) = self.get_best_quote(
            token_in, token_out, amount_after_fee
        )
        router = best_router
    
    assert router != empty(address), "No router available"
    
    # Approve router
    IERC20(token_in).approve(router, amount_after_fee)
    
    # Execute swap
    path: DynArray[address, 5] = [token_in, token_out]
    
    amounts: DynArray[uint256, 5] = IUniswapV2Router(router).swapExactTokensForTokens(
        amount_after_fee,
        min_amount_out,
        path,
        msg.sender,
        block.timestamp + 300  # 5 min deadline
    )
    
    amount_out: uint256 = amounts[len(amounts) - 1]
    assert amount_out >= min_amount_out, "Slippage too high"
    
    log Swapped(msg.sender, token_in, token_out, amount_in, amount_out, router)
    
    return amount_out

@external
@payable
@nonreentrant
def swap_eth_for_tokens(
    token_out: address,
    min_amount_out: uint256
) -> uint256:
    """Swap ETH เป็น Token"""
    assert not self.is_paused, "Paused"
    assert msg.value > 0, "Send ETH"
    
    fee: uint256 = msg.value * PROTOCOL_FEE_BPS / 10000
    amount_after_fee: uint256 = msg.value - fee
    self.collected_fees[empty(address)] += fee
    
    # หา best router
    path: DynArray[address, 5] = [self.weth, token_out]
    best_router: address = empty(address)
    best_amount: uint256 = 0
    
    for router_info: RouterInfo in self.routers:
        if not router_info.active or router_info.router_type != 0:
            continue
        
        quote: uint256 = self._get_v2_quote(
            router_info.router_address, amount_after_fee, path
        )
        if quote > best_amount:
            best_amount = quote
            best_router = router_info.router_address
    
    assert best_router != empty(address), "No router"
    
    amounts: DynArray[uint256, 5] = IUniswapV2Router(best_router).swapExactETHForTokens(
        min_amount_out,
        path,
        msg.sender,
        block.timestamp + 300,
        value=amount_after_fee
    )
    
    amount_out: uint256 = amounts[len(amounts) - 1]
    
    log Swapped(msg.sender, empty(address), token_out, msg.value, amount_out, best_router)
    
    return amount_out

@external
@nonreentrant
def swap_tokens_for_eth(
    token_in: address,
    amount_in: uint256,
    min_eth_out: uint256
) -> uint256:
    """Swap Token เป็น ETH"""
    assert not self.is_paused, "Paused"
    assert amount_in > 0, "Must swap > 0"
    
    IERC20(token_in).transferFrom(msg.sender, self, amount_in)
    
    fee: uint256 = amount_in * PROTOCOL_FEE_BPS / 10000
    amount_after_fee: uint256 = amount_in - fee
    self.collected_fees[token_in] += fee
    
    path: DynArray[address, 5] = [token_in, self.weth]
    best_router: address = empty(address)
    best_amount: uint256 = 0
    
    for router_info: RouterInfo in self.routers:
        if not router_info.active or router_info.router_type != 0:
            continue
        
        quote: uint256 = self._get_v2_quote(
            router_info.router_address, amount_after_fee, path
        )
        if quote > best_amount:
            best_amount = quote
            best_router = router_info.router_address
    
    assert best_router != empty(address), "No router"
    
    IERC20(token_in).approve(best_router, amount_after_fee)
    
    amounts: DynArray[uint256, 5] = IUniswapV2Router(best_router).swapExactTokensForETH(
        amount_after_fee,
        min_eth_out,
        path,
        msg.sender,
        block.timestamp + 300
    )
    
    eth_out: uint256 = amounts[len(amounts) - 1]
    
    log Swapped(msg.sender, token_in, empty(address), amount_in, eth_out, best_router)
    
    return eth_out

# ==================== Fee Management ====================

@external
def collect_fees(token: address):
    """เก็บ protocol fees"""
    assert msg.sender == self.owner, "Not owner"
    
    amount: uint256 = self.collected_fees[token]
    assert amount > 0, "No fees"
    
    self.collected_fees[token] = 0
    
    if token == empty(address):
        send(self.fee_recipient, amount)
    else:
        IERC20(token).transfer(self.fee_recipient, amount)

# ==================== Admin ====================

@external
def set_paused(state: bool):
    assert msg.sender == self.owner
    self.is_paused = state

@external
def set_fee_recipient(recipient: address):
    assert msg.sender == self.owner
    assert recipient != empty(address)
    self.fee_recipient = recipient

@external
@payable
def __default__():
    """รับ ETH"""
    pass
```

---

## Test Code

```python
# tests/test_cross_contract.py
import pytest
from ape import accounts, project

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def user(accounts):
    return accounts[1]

@pytest.fixture
def mock_token_a(owner, project):
    return project.MockERC20.deploy("Token A", "TKNA", 18, sender=owner)

@pytest.fixture
def mock_token_b(owner, project):
    return project.MockERC20.deploy("Token B", "TKNB", 18, sender=owner)

@pytest.fixture
def weth(owner, project):
    return project.MockWETH.deploy(sender=owner)

@pytest.fixture
def mock_router(owner, project, mock_token_a, mock_token_b, weth):
    router = project.MockUniswapV2Router.deploy(
        weth.address,
        sender=owner
    )
    # Setup price: 1 TKNA = 2 TKNB
    router.setAmountOut(mock_token_a.address, mock_token_b.address, 2 * 10**18)
    return router

@pytest.fixture
def aggregator(owner, weth, project):
    return project.DeFiAggregator.deploy(
        weth.address,
        owner.address,
        sender=owner
    )

class TestAggregatorSetup:
    
    def test_add_router(self, aggregator, mock_router, owner):
        """ทดสอบเพิ่ม router"""
        aggregator.add_router(
            mock_router.address,
            "Uniswap V2",
            0,  # V2 type
            sender=owner
        )
        
        assert aggregator.router_count() == 1
    
    def test_only_owner_can_add(self, aggregator, mock_router, user):
        """ทดสอบว่าแค่ owner เพิ่ม router ได้"""
        with pytest.raises(Exception):
            aggregator.add_router(
                mock_router.address,
                "Uniswap V2",
                0,
                sender=user
            )

class TestQuotes:
    
    def test_get_best_quote(
        self, aggregator, mock_router, mock_token_a, mock_token_b, owner
    ):
        """ทดสอบดึง best quote"""
        aggregator.add_router(mock_router.address, "Router1", 0, sender=owner)
        
        best_router, best_amount = aggregator.get_best_quote(
            mock_token_a.address,
            mock_token_b.address,
            1 * 10**18  # 1 TKNA
        )
        
        assert best_router == mock_router.address
        assert best_amount == 2 * 10**18  # 2 TKNB

class TestSwaps:
    
    def test_swap_tokens(
        self, aggregator, mock_router, mock_token_a, mock_token_b, 
        owner, user
    ):
        """ทดสอบ swap tokens"""
        aggregator.add_router(mock_router.address, "Router1", 0, sender=owner)
        
        # Mint tokens ให้ user
        amount = 10 * 10**18
        mock_token_a.mint(user.address, amount, sender=owner)
        mock_token_a.approve(aggregator.address, amount, sender=user)
        
        # Mock router ต้องมี TKNB เพื่อส่งกลับ
        mock_token_b.mint(mock_router.address, 100 * 10**18, sender=owner)
        
        balance_before = mock_token_b.balanceOf(user.address)
        
        aggregator.swap_tokens(
            mock_token_a.address,
            mock_token_b.address,
            amount,
            1,  # min out
            empty(address),  # auto select
            sender=user
        )
        
        balance_after = mock_token_b.balanceOf(user.address)
        assert balance_after > balance_before

class TestFees:
    
    def test_fee_collection(
        self, aggregator, mock_router, mock_token_a, mock_token_b,
        owner, user
    ):
        """ทดสอบการเก็บ fee"""
        aggregator.add_router(mock_router.address, "Router1", 0, sender=owner)
        
        amount = 1000 * 10**18
        mock_token_a.mint(user.address, amount, sender=owner)
        mock_token_a.approve(aggregator.address, amount, sender=user)
        mock_token_b.mint(mock_router.address, 1000000 * 10**18, sender=owner)
        
        aggregator.swap_tokens(
            mock_token_a.address,
            mock_token_b.address,
            amount, 1, empty(address),
            sender=user
        )
        
        # ตรวจสอบ fee
        fee = aggregator.collected_fees(mock_token_a.address)
        expected_fee = amount * 10 // 10000  # 0.1%
        assert fee == expected_fee
        
        # เก็บ fee
        owner_balance_before = mock_token_a.balanceOf(owner.address)
        aggregator.collect_fees(mock_token_a.address, sender=owner)
        owner_balance_after = mock_token_a.balanceOf(owner.address)
        
        assert owner_balance_after - owner_balance_before == expected_fee
```

---

## สรุป

Cross-contract calls ใน Vyper มีสองแบบหลัก:

| วิธี | ใช้เมื่อ | ข้อดี | ข้อเสีย |
|------|---------|-------|---------|
| Interface | รู้ ABI ล่วงหน้า | Type safe, ชัดเจน | ต้องกำหนด Interface |
| raw_call | Dynamic, ไม่รู้ ABI | Flexible | เสี่ยงถ้าไม่ระวัง |

### Security Best Practices
- [ ] ใช้ `@nonreentrant` เสมอเมื่อ call external contract
- [ ] ตรวจสอบ return value
- [ ] ใช้ CEI pattern (Checks → Effects → Interactions)
- [ ] Validate address ก่อน call
- [ ] Set deadline สำหรับ swap

---

[⬅️ Part 038: Oracle Integration](part_038_oracle.md) | [Part 040: Proxy Patterns ➡️](part_040_proxy.md)
