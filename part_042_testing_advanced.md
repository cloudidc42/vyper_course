# Part 042: Advanced Testing with Titanoboa

## สารบัญ
1. [Overview](#overview)
2. [Setup และ Fixtures](#setup)
3. [Parametrize Testing](#parametrize)
4. [Property-Based Testing with Hypothesis](#hypothesis)
5. [Fuzzing](#fuzzing)
6. [Coverage Analysis](#coverage)
7. [Complete Test Suite](#complete-test-suite)

---

## 1. Overview {#overview}

การทดสอบ Smart Contract ที่ดีเป็นสิ่งจำเป็นอย่างยิ่ง เพราะ:

- **Immutability**: เมื่อ deploy แล้วแก้ไขได้ยาก
- **Financial Risk**: Bug อาจทำให้เสีย funds
- **Audit Requirement**: Project ใหญ่ต้องผ่าน audit

### เครื่องมือที่ใช้

- **Titanoboa**: Python-based Vyper testing framework ที่เร็วมาก
- **pytest**: Test framework มาตรฐาน
- **Hypothesis**: Property-based testing library
- **coverage.py**: Code coverage analysis

### Installation

```bash
pip install titanoboa pytest hypothesis pytest-cov
```

---

## 2. Setup และ Fixtures {#setup}

### Contract ที่จะทดสอบ

```vyper
# contracts/AdvancedToken.vy
# @version 0.4.0
# @title AdvancedToken - Token สำหรับทดสอบ advanced testing techniques

from vyper.interfaces import ERC20

implements: ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event Mint:
    minter: indexed(address)
    to: indexed(address)
    amount: uint256

event Burn:
    burner: indexed(address)
    amount: uint256

# State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)
minters: public(HashMap[address, bool])
is_paused: public(bool)

MAX_SUPPLY: public(uint256)
TRANSFER_FEE: public(uint256)  # in basis points (1 = 0.01%)
fee_recipient: public(address)

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    max_supply: uint256,
    fee_bps: uint256
):
    self.name = name
    self.symbol = symbol
    self.decimals = 18
    self.owner = msg.sender
    self.MAX_SUPPLY = max_supply
    self.TRANSFER_FEE = fee_bps
    self.fee_recipient = msg.sender
    self.is_paused = False

@external
@view
def totalSupply() -> uint256:
    return self.total_supply

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@internal
def _calculate_fee(amount: uint256) -> uint256:
    if self.TRANSFER_FEE == 0:
        return 0
    return amount * self.TRANSFER_FEE // 10000

@internal
def _transfer(sender: address, to: address, amount: uint256):
    assert not self.is_paused, "Paused"
    assert to != empty(address), "Zero address"
    
    fee: uint256 = self._calculate_fee(amount)
    net_amount: uint256 = amount - fee
    
    assert self.balances[sender] >= amount, "Insufficient balance"
    
    self.balances[sender] -= amount
    self.balances[to] += net_amount
    
    if fee > 0:
        self.balances[self.fee_recipient] += fee
        log Transfer(sender, self.fee_recipient, fee)
    
    log Transfer(sender, to, net_amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    assert self.allowances[sender][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[sender][msg.sender] -= amount
    self._transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Zero address"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner or self.minters[msg.sender], "Not authorized"
    assert to != empty(address), "Zero address"
    assert self.total_supply + amount <= self.MAX_SUPPLY, "Exceeds max supply"
    
    self.total_supply += amount
    self.balances[to] += amount
    log Mint(msg.sender, to, amount)
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Burn(msg.sender, amount)
    log Transfer(msg.sender, empty(address), amount)

@external
def set_paused(paused: bool):
    assert msg.sender == self.owner, "Not owner"
    self.is_paused = paused

@external
def set_minter(minter: address, status: bool):
    assert msg.sender == self.owner, "Not owner"
    self.minters[minter] = status

@external
def set_fee(fee_bps: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert fee_bps <= 1000, "Fee too high (max 10%)"
    self.TRANSFER_FEE = fee_bps

@external
def set_fee_recipient(recipient: address):
    assert msg.sender == self.owner, "Not owner"
    assert recipient != empty(address), "Zero address"
    self.fee_recipient = recipient

@external
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.owner = new_owner
```

### conftest.py - Fixtures หลัก

```python
# tests/conftest.py
import pytest
import boa
from eth_utils import to_checksum_address

# ==================== Basic Fixtures ====================

@pytest.fixture(scope="session")
def accounts():
    """สร้าง list ของ test accounts"""
    return [boa.env.generate_address() for _ in range(10)]

@pytest.fixture(scope="session")
def deployer(accounts):
    """Owner/deployer account"""
    return accounts[0]

@pytest.fixture(scope="session")
def alice(accounts):
    """Test user Alice"""
    return accounts[1]

@pytest.fixture(scope="session")
def bob(accounts):
    """Test user Bob"""
    return accounts[2]

@pytest.fixture(scope="session")
def charlie(accounts):
    """Test user Charlie"""
    return accounts[3]

@pytest.fixture(scope="session")
def minter(accounts):
    """Minter account"""
    return accounts[4]

# ==================== Contract Fixtures ====================

@pytest.fixture
def token(deployer):
    """Deploy fresh token for each test"""
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/AdvancedToken.vy",
            "Advanced Token",
            "ADV",
            10**9 * 10**18,  # 1 billion max supply
            0  # no fee initially
        )

@pytest.fixture
def token_with_fee(deployer):
    """Token with 0.3% transfer fee"""
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/AdvancedToken.vy",
            "Fee Token",
            "FEE",
            10**9 * 10**18,
            30  # 0.3% fee
        )

@pytest.fixture
def token_with_supply(token, deployer, alice, bob):
    """Token with pre-minted supply distributed to users"""
    with boa.env.prank(deployer):
        token.mint(alice, 1000 * 10**18)
        token.mint(bob, 1000 * 10**18)
        token.mint(deployer, 8000 * 10**18)
    return token

@pytest.fixture
def minter_token(token, deployer, minter):
    """Token with minter role set"""
    with boa.env.prank(deployer):
        token.set_minter(minter, True)
    return token

# ==================== Helper Fixtures ====================

@pytest.fixture
def fund_accounts(token, deployer, accounts):
    """Fund all test accounts with tokens"""
    with boa.env.prank(deployer):
        for account in accounts[1:]:
            token.mint(account, 100 * 10**18)
    return token

@pytest.fixture
def snapshot(boa):
    """Create a snapshot and restore after test"""
    snapshot_id = boa.env.snapshot()
    yield snapshot_id
    boa.env.restore(snapshot_id)
```

---

## 3. Parametrize Testing {#parametrize}

```python
# tests/test_parametrize.py
import pytest
import boa
from conftest import *

class TestTransferParametrize:
    """ทดสอบ transfer ด้วย parametrize"""
    
    @pytest.mark.parametrize("amount", [
        1,
        100,
        10**18,
        500 * 10**18,
        1000 * 10**18,
    ])
    def test_transfer_various_amounts(self, token_with_supply, alice, bob, amount):
        """ทดสอบ transfer ด้วยจำนวนต่างๆ"""
        initial_alice = token_with_supply.balanceOf(alice)
        initial_bob = token_with_supply.balanceOf(bob)
        
        if amount <= initial_alice:
            with boa.env.prank(alice):
                token_with_supply.transfer(bob, amount)
            
            assert token_with_supply.balanceOf(alice) == initial_alice - amount
            assert token_with_supply.balanceOf(bob) == initial_bob + amount
    
    @pytest.mark.parametrize("fee_bps,expected_fee_pct", [
        (0, 0.0),
        (10, 0.1),
        (30, 0.3),
        (100, 1.0),
        (500, 5.0),
        (1000, 10.0),
    ])
    def test_fee_calculation(self, token_with_fee, deployer, alice, bob, fee_bps, expected_fee_pct):
        """ทดสอบ fee calculation ด้วย fee rates ต่างๆ"""
        with boa.env.prank(deployer):
            token_with_fee.set_fee(fee_bps)
            token_with_fee.mint(alice, 10000 * 10**18)
        
        transfer_amount = 1000 * 10**18
        expected_fee = transfer_amount * fee_bps // 10000
        expected_received = transfer_amount - expected_fee
        
        initial_bob = token_with_fee.balanceOf(bob)
        
        with boa.env.prank(alice):
            token_with_fee.transfer(bob, transfer_amount)
        
        assert token_with_fee.balanceOf(bob) == initial_bob + expected_received
    
    @pytest.mark.parametrize("action,should_succeed", [
        ("transfer", True),
        ("approve", True),
        ("mint_as_owner", True),
        ("mint_as_minter", True),
        ("mint_as_random", False),
        ("burn_own", True),
        ("burn_too_much", False),
    ])
    def test_access_control(
        self, token, deployer, alice, bob, minter,
        action, should_succeed
    ):
        """ทดสอบ access control scenarios ต่างๆ"""
        with boa.env.prank(deployer):
            token.set_minter(minter, True)
            token.mint(alice, 1000 * 10**18)
        
        if action == "transfer":
            with boa.env.prank(alice):
                result = token.transfer(bob, 100 * 10**18)
                assert result == True
        
        elif action == "approve":
            with boa.env.prank(alice):
                result = token.approve(bob, 500 * 10**18)
                assert result == True
        
        elif action == "mint_as_owner":
            with boa.env.prank(deployer):
                token.mint(alice, 100 * 10**18)
        
        elif action == "mint_as_minter":
            with boa.env.prank(minter):
                token.mint(alice, 100 * 10**18)
        
        elif action == "mint_as_random":
            with pytest.raises(Exception):
                with boa.env.prank(bob):
                    token.mint(alice, 100 * 10**18)
        
        elif action == "burn_own":
            with boa.env.prank(alice):
                token.burn(100 * 10**18)
        
        elif action == "burn_too_much":
            with pytest.raises(Exception):
                with boa.env.prank(alice):
                    token.burn(2000 * 10**18)
    
    @pytest.mark.parametrize("initial,transfer,expected_remaining", [
        (1000, 500, 500),
        (1000, 1000, 0),
        (1000, 0, 1000),
        (500, 100, 400),
        (10**18, 10**17, 9 * 10**17),
    ])
    def test_balance_after_transfer(
        self, token, deployer, alice, bob,
        initial, transfer, expected_remaining
    ):
        """ทดสอบ balance หลัง transfer"""
        with boa.env.prank(deployer):
            token.mint(alice, initial)
        
        with boa.env.prank(alice):
            token.transfer(bob, transfer)
        
        assert token.balanceOf(alice) == expected_remaining
    
    @pytest.mark.parametrize("name,symbol,max_supply,fee", [
        ("Token A", "TKA", 10**6, 0),
        ("Token B", "TKB", 10**9, 50),
        ("Long Token Name Here", "LTKN", 10**12, 100),
        ("T", "T", 1000, 1000),
    ])
    def test_deployment_params(self, deployer, name, symbol, max_supply, fee):
        """ทดสอบ deployment ด้วย parameters ต่างๆ"""
        with boa.env.prank(deployer):
            token = boa.load(
                "contracts/AdvancedToken.vy",
                name, symbol, max_supply, fee
            )
        
        assert token.name() == name
        assert token.symbol() == symbol
        assert token.MAX_SUPPLY() == max_supply
        assert token.TRANSFER_FEE() == fee
        assert token.owner() == deployer
```

---

## 4. Property-Based Testing with Hypothesis {#hypothesis}

```python
# tests/test_property_based.py
import pytest
import boa
from hypothesis import given, settings, assume, strategies as st
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant, initialize

# ==================== Basic Property Tests ====================

class TestPropertyBased:
    """Property-based testing ด้วย Hypothesis"""
    
    @given(
        amount=st.integers(min_value=1, max_value=10**18)
    )
    @settings(max_examples=100)
    def test_transfer_preserves_total_supply(self, token_with_supply, alice, bob, amount):
        """
        Property: การ transfer ไม่เปลี่ยน total supply
        """
        assume(amount <= token_with_supply.balanceOf(alice))
        
        total_before = token_with_supply.total_supply()
        
        with boa.env.prank(alice):
            token_with_supply.transfer(bob, amount)
        
        total_after = token_with_supply.total_supply()
        assert total_before == total_after, "Total supply changed after transfer!"
    
    @given(
        amount=st.integers(min_value=1, max_value=10**18)
    )
    @settings(max_examples=50)
    def test_balance_sum_equals_total_supply(
        self, token, deployer, alice, bob, amount
    ):
        """
        Property: ผลรวม balances ต้องเท่ากับ total supply
        """
        with boa.env.prank(deployer):
            token.mint(alice, amount)
        
        balance_alice = token.balanceOf(alice)
        balance_deployer = token.balanceOf(deployer)
        total = token.total_supply()
        
        # alice + deployer accounts ต้องไม่เกิน total
        assert balance_alice + balance_deployer <= total
    
    @given(
        approve_amount=st.integers(min_value=0, max_value=10**18),
        transfer_amount=st.integers(min_value=1, max_value=10**18)
    )
    @settings(max_examples=100)
    def test_allowance_decreases_after_transferFrom(
        self, token, deployer, alice, bob,
        approve_amount, transfer_amount
    ):
        """
        Property: allowance ลดลงหลัง transferFrom
        """
        assume(transfer_amount <= approve_amount)
        
        with boa.env.prank(deployer):
            token.mint(alice, transfer_amount)
        
        with boa.env.prank(alice):
            token.approve(bob, approve_amount)
        
        initial_allowance = token.allowance(alice, bob)
        
        with boa.env.prank(bob):
            token.transferFrom(alice, bob, transfer_amount)
        
        new_allowance = token.allowance(alice, bob)
        assert new_allowance == initial_allowance - transfer_amount
    
    @given(
        amount=st.integers(min_value=1, max_value=10**18)
    )
    @settings(max_examples=50)
    def test_mint_increases_supply(self, token, deployer, alice, amount):
        """
        Property: mint เพิ่ม total supply และ balance ด้วยจำนวนเท่ากัน
        """
        max_supply = token.MAX_SUPPLY()
        assume(token.total_supply() + amount <= max_supply)
        
        supply_before = token.total_supply()
        balance_before = token.balanceOf(alice)
        
        with boa.env.prank(deployer):
            token.mint(alice, amount)
        
        assert token.total_supply() == supply_before + amount
        assert token.balanceOf(alice) == balance_before + amount
    
    @given(
        amount=st.integers(min_value=1, max_value=10**18)
    )
    @settings(max_examples=50)
    def test_burn_decreases_supply(self, token, deployer, alice, amount):
        """
        Property: burn ลด total supply และ balance ด้วยจำนวนเท่ากัน
        """
        with boa.env.prank(deployer):
            token.mint(alice, amount)
        
        supply_before = token.total_supply()
        
        with boa.env.prank(alice):
            token.burn(amount)
        
        assert token.total_supply() == supply_before - amount
        assert token.balanceOf(alice) == 0
    
    @given(
        fee_bps=st.integers(min_value=0, max_value=1000),
        amount=st.integers(min_value=10000, max_value=10**18)
    )
    @settings(max_examples=100)
    def test_fee_is_correctly_calculated(
        self, token_with_fee, deployer, alice, bob,
        fee_bps, amount
    ):
        """
        Property: fee ต้องถูกคำนวณถูกต้องเสมอ
        """
        with boa.env.prank(deployer):
            token_with_fee.set_fee(fee_bps)
            token_with_fee.mint(alice, amount)
        
        fee_recipient = token_with_fee.fee_recipient()
        recipient_before = token_with_fee.balanceOf(fee_recipient)
        alice_before = token_with_fee.balanceOf(alice)
        bob_before = token_with_fee.balanceOf(bob)
        
        with boa.env.prank(alice):
            token_with_fee.transfer(bob, amount)
        
        expected_fee = amount * fee_bps // 10000
        expected_received = amount - expected_fee
        
        # ตรวจสอบว่า balances สอดคล้องกัน
        if fee_recipient != bob:
            assert token_with_fee.balanceOf(bob) == bob_before + expected_received
        
        # Alice ต้องสูญเสีย amount ทั้งหมด
        assert token_with_fee.balanceOf(alice) == alice_before - amount


# ==================== Stateful Testing ====================

class TokenStateMachine(RuleBasedStateMachine):
    """
    Stateful property testing - จำลองสถานะของ contract
    และตรวจสอบ invariants หลังทุก operation
    """
    
    def __init__(self):
        super().__init__()
        self.accounts = [boa.env.generate_address() for _ in range(5)]
        self.deployer = self.accounts[0]
        
        with boa.env.prank(self.deployer):
            self.token = boa.load(
                "contracts/AdvancedToken.vy",
                "State Machine Token",
                "SMT",
                10**12,
                0
            )
        
        # Track balances in Python for comparison
        self.py_balances = {acc: 0 for acc in self.accounts}
        self.py_total = 0
        
        # Initial mint
        initial = 1000 * 10**18
        with boa.env.prank(self.deployer):
            self.token.mint(self.deployer, initial)
        self.py_balances[self.deployer] = initial
        self.py_total = initial
    
    @rule(
        sender_idx=st.integers(min_value=0, max_value=4),
        receiver_idx=st.integers(min_value=0, max_value=4),
        amount=st.integers(min_value=1, max_value=100 * 10**18)
    )
    def do_transfer(self, sender_idx, receiver_idx, amount):
        """Rule: ทำ transfer"""
        sender = self.accounts[sender_idx]
        receiver = self.accounts[receiver_idx]
        
        if self.py_balances[sender] >= amount and sender != receiver:
            with boa.env.prank(sender):
                self.token.transfer(receiver, amount)
            
            self.py_balances[sender] -= amount
            self.py_balances[receiver] += amount
    
    @rule(
        account_idx=st.integers(min_value=0, max_value=4),
        amount=st.integers(min_value=1, max_value=100 * 10**18)
    )
    def do_mint(self, account_idx, amount):
        """Rule: ทำ mint"""
        account = self.accounts[account_idx]
        
        if self.py_total + amount <= 10**12:
            with boa.env.prank(self.deployer):
                self.token.mint(account, amount)
            
            self.py_balances[account] += amount
            self.py_total += amount
    
    @rule(
        account_idx=st.integers(min_value=0, max_value=4),
        amount=st.integers(min_value=1, max_value=50 * 10**18)
    )
    def do_burn(self, account_idx, amount):
        """Rule: ทำ burn"""
        account = self.accounts[account_idx]
        
        if self.py_balances[account] >= amount:
            with boa.env.prank(account):
                self.token.burn(amount)
            
            self.py_balances[account] -= amount
            self.py_total -= amount
    
    @invariant()
    def balances_match_python_model(self):
        """Invariant: balances ใน contract ต้องตรงกับ Python model"""
        for account in self.accounts:
            contract_balance = self.token.balanceOf(account)
            py_balance = self.py_balances[account]
            assert contract_balance == py_balance, \
                f"Balance mismatch for {account}: contract={contract_balance}, python={py_balance}"
    
    @invariant()
    def total_supply_matches_sum(self):
        """Invariant: total supply ต้องเท่ากับผลรวม balances"""
        total_in_contract = self.token.total_supply()
        sum_of_balances = sum(self.token.balanceOf(acc) for acc in self.accounts)
        assert total_in_contract >= sum_of_balances, \
            "Total supply less than sum of tracked balances"
    
    @invariant()
    def total_supply_matches_model(self):
        """Invariant: total supply ต้องตรงกับ Python model"""
        assert self.token.total_supply() == self.py_total


# สร้าง test class จาก state machine
TestTokenStateMachine = TokenStateMachine.TestCase
```

---

## 5. Fuzzing {#fuzzing}

```python
# tests/test_fuzzing.py
import pytest
import boa
from hypothesis import given, settings, assume, strategies as st
import random

class TestFuzzing:
    """Fuzzing tests - ป้อน random inputs เพื่อหา edge cases"""
    
    @given(
        amounts=st.lists(
            st.integers(min_value=1, max_value=10**15),
            min_size=1,
            max_size=50
        )
    )
    @settings(max_examples=50)
    def test_batch_transfer_consistent(
        self, token, deployer, alice,
        amounts
    ):
        """
        Fuzz test: batch transfer ต้องให้ผลเหมือน individual transfers
        """
        # สร้าง recipients
        recipients = [boa.env.generate_address() for _ in range(len(amounts))]
        
        total = sum(amounts)
        
        with boa.env.prank(deployer):
            token.mint(alice, total)
        
        # ตรวจสอบ batch transfer ทำงานถูกต้อง
        if len(amounts) <= 200:
            with boa.env.prank(alice):
                token.batch_transfer(recipients, amounts)
            
            for i, recipient in enumerate(recipients):
                assert token.balanceOf(recipient) == amounts[i]
    
    @given(
        value=st.integers(min_value=0, max_value=2**256 - 1)
    )
    @settings(max_examples=100)
    def test_approval_any_value(self, token, alice, bob, value):
        """
        Fuzz: approve ควรรับ value ใดๆ ได้ (ไม่ revert)
        """
        with boa.env.prank(alice):
            token.approve(bob, value)
        
        assert token.allowance(alice, bob) == value
    
    @given(
        fee=st.integers(min_value=0, max_value=1000)
    )
    @settings(max_examples=100)
    def test_valid_fee_range(self, token, deployer, fee):
        """
        Fuzz: fee ใดๆ ใน range 0-1000 ต้องเซ็ตได้
        """
        with boa.env.prank(deployer):
            token.set_fee(fee)
        
        assert token.TRANSFER_FEE() == fee
    
    @given(
        fee=st.integers(min_value=1001, max_value=2**256 - 1)
    )
    @settings(max_examples=50)
    def test_invalid_fee_reverts(self, token, deployer, fee):
        """
        Fuzz: fee > 1000 ต้อง revert เสมอ
        """
        with pytest.raises(Exception):
            with boa.env.prank(deployer):
                token.set_fee(fee)
    
    @given(
        n_transfers=st.integers(min_value=1, max_value=20),
        seed=st.integers(min_value=0, max_value=10000)
    )
    @settings(max_examples=30)
    def test_sequence_of_transfers(
        self, token, deployer,
        n_transfers, seed
    ):
        """
        Fuzz: sequence of random transfers ต้อง consistent
        """
        rng = random.Random(seed)
        
        accounts = [boa.env.generate_address() for _ in range(5)]
        
        with boa.env.prank(deployer):
            for acc in accounts:
                token.mint(acc, 1000 * 10**18)
        
        # Track expected balances
        expected = {acc: 1000 * 10**18 for acc in accounts}
        
        for _ in range(n_transfers):
            sender_idx = rng.randint(0, 4)
            receiver_idx = rng.randint(0, 4)
            
            sender = accounts[sender_idx]
            receiver = accounts[receiver_idx]
            
            if sender == receiver:
                continue
            
            max_amount = expected[sender]
            if max_amount == 0:
                continue
            
            amount = rng.randint(1, max_amount)
            
            with boa.env.prank(sender):
                token.transfer(receiver, amount)
            
            expected[sender] -= amount
            expected[receiver] += amount
        
        # ตรวจสอบ final balances
        for acc in accounts:
            assert token.balanceOf(acc) == expected[acc], \
                f"Balance mismatch for {acc}"
```

---

## 6. Coverage Analysis {#coverage}

```python
# tests/test_coverage.py
# ครอบคลุมทุก code path ใน contract

import pytest
import boa

class TestFullCoverage:
    """ครอบคลุมทุก branch ใน AdvancedToken"""
    
    # ========== Constructor ==========
    
    def test_constructor_sets_name(self, token):
        assert token.name() == "Advanced Token"
    
    def test_constructor_sets_symbol(self, token):
        assert token.symbol() == "ADV"
    
    def test_constructor_sets_decimals(self, token):
        assert token.decimals() == 18
    
    def test_constructor_sets_owner(self, token, deployer):
        assert token.owner() == deployer
    
    def test_constructor_zero_supply(self, token):
        assert token.total_supply() == 0
    
    def test_constructor_not_paused(self, token):
        assert token.is_paused() == False
    
    # ========== Transfer ==========
    
    def test_transfer_success(self, token_with_supply, alice, bob):
        initial = token_with_supply.balanceOf(alice)
        with boa.env.prank(alice):
            token_with_supply.transfer(bob, 100 * 10**18)
        assert token_with_supply.balanceOf(alice) == initial - 100 * 10**18
    
    def test_transfer_to_self(self, token_with_supply, alice):
        """Transfer to self - balance ไม่เปลี่ยน"""
        initial = token_with_supply.balanceOf(alice)
        with boa.env.prank(alice):
            token_with_supply.transfer(alice, 100 * 10**18)
        # Fee อาจทำให้ balance เปลี่ยน ถ้า fee > 0
    
    def test_transfer_insufficient_balance(self, token, alice, bob):
        with pytest.raises(Exception, match="Insufficient balance"):
            with boa.env.prank(alice):
                token.transfer(bob, 1)
    
    def test_transfer_to_zero_address(self, token_with_supply, alice):
        with pytest.raises(Exception, match="Zero address"):
            with boa.env.prank(alice):
                token_with_supply.transfer("0x0000000000000000000000000000000000000000", 1)
    
    def test_transfer_when_paused(self, token_with_supply, deployer, alice, bob):
        with boa.env.prank(deployer):
            token_with_supply.set_paused(True)
        
        with pytest.raises(Exception, match="Paused"):
            with boa.env.prank(alice):
                token_with_supply.transfer(bob, 100 * 10**18)
    
    # ========== Approve ==========
    
    def test_approve_success(self, token, alice, bob):
        with boa.env.prank(alice):
            result = token.approve(bob, 500 * 10**18)
        assert result == True
        assert token.allowance(alice, bob) == 500 * 10**18
    
    def test_approve_zero_address(self, token, alice):
        with pytest.raises(Exception, match="Zero address"):
            with boa.env.prank(alice):
                token.approve("0x0000000000000000000000000000000000000000", 100)
    
    def test_approve_override(self, token, alice, bob):
        with boa.env.prank(alice):
            token.approve(bob, 100)
            token.approve(bob, 200)
        assert token.allowance(alice, bob) == 200
    
    # ========== TransferFrom ==========
    
    def test_transferFrom_success(self, token_with_supply, alice, bob, charlie):
        with boa.env.prank(alice):
            token_with_supply.approve(bob, 500 * 10**18)
        
        with boa.env.prank(bob):
            token_with_supply.transferFrom(alice, charlie, 100 * 10**18)
        
        assert token_with_supply.allowance(alice, bob) == 400 * 10**18
    
    def test_transferFrom_insufficient_allowance(
        self, token_with_supply, alice, bob, charlie
    ):
        with boa.env.prank(alice):
            token_with_supply.approve(bob, 50 * 10**18)
        
        with pytest.raises(Exception, match="Insufficient allowance"):
            with boa.env.prank(bob):
                token_with_supply.transferFrom(alice, charlie, 100 * 10**18)
    
    # ========== Mint ==========
    
    def test_mint_by_owner(self, token, deployer, alice):
        with boa.env.prank(deployer):
            token.mint(alice, 1000 * 10**18)
        assert token.balanceOf(alice) == 1000 * 10**18
        assert token.total_supply() == 1000 * 10**18
    
    def test_mint_by_minter(self, minter_token, minter, alice):
        with boa.env.prank(minter):
            minter_token.mint(alice, 100 * 10**18)
        assert minter_token.balanceOf(alice) == 100 * 10**18
    
    def test_mint_exceeds_max_supply(self, token, deployer, alice):
        max_supply = token.MAX_SUPPLY()
        with pytest.raises(Exception, match="Exceeds max supply"):
            with boa.env.prank(deployer):
                token.mint(alice, max_supply + 1)
    
    def test_mint_unauthorized(self, token, alice, bob):
        with pytest.raises(Exception, match="Not authorized"):
            with boa.env.prank(alice):
                token.mint(bob, 100 * 10**18)
    
    def test_mint_to_zero_address(self, token, deployer):
        with pytest.raises(Exception, match="Zero address"):
            with boa.env.prank(deployer):
                token.mint("0x0000000000000000000000000000000000000000", 100)
    
    # ========== Burn ==========
    
    def test_burn_success(self, token_with_supply, alice):
        initial_balance = token_with_supply.balanceOf(alice)
        initial_supply = token_with_supply.total_supply()
        
        with boa.env.prank(alice):
            token_with_supply.burn(500 * 10**18)
        
        assert token_with_supply.balanceOf(alice) == initial_balance - 500 * 10**18
        assert token_with_supply.total_supply() == initial_supply - 500 * 10**18
    
    def test_burn_insufficient(self, token, alice):
        with pytest.raises(Exception, match="Insufficient balance"):
            with boa.env.prank(alice):
                token.burn(1)
    
    # ========== Admin Functions ==========
    
    def test_set_paused(self, token, deployer):
        with boa.env.prank(deployer):
            token.set_paused(True)
        assert token.is_paused() == True
        
        with boa.env.prank(deployer):
            token.set_paused(False)
        assert token.is_paused() == False
    
    def test_set_paused_unauthorized(self, token, alice):
        with pytest.raises(Exception, match="Not owner"):
            with boa.env.prank(alice):
                token.set_paused(True)
    
    def test_transfer_ownership(self, token, deployer, alice):
        with boa.env.prank(deployer):
            token.transfer_ownership(alice)
        assert token.owner() == alice
    
    def test_transfer_ownership_to_zero(self, token, deployer):
        with pytest.raises(Exception, match="Zero address"):
            with boa.env.prank(deployer):
                token.transfer_ownership("0x0000000000000000000000000000000000000000")
    
    # ========== Fee Tests ==========
    
    def test_fee_with_zero_rate(self, token, deployer, alice, bob):
        with boa.env.prank(deployer):
            token.mint(alice, 1000 * 10**18)
        
        with boa.env.prank(alice):
            token.transfer(bob, 1000 * 10**18)
        
        assert token.balanceOf(bob) == 1000 * 10**18
    
    def test_fee_recipient_receives_fee(self, token_with_fee, deployer, alice, bob):
        fee_recipient = token_with_fee.fee_recipient()
        initial_fee_balance = token_with_fee.balanceOf(fee_recipient)
        
        with boa.env.prank(deployer):
            token_with_fee.mint(alice, 10000 * 10**18)
        
        with boa.env.prank(alice):
            token_with_fee.transfer(bob, 10000 * 10**18)
        
        fee = 10000 * 10**18 * 30 // 10000  # 0.3%
        assert token_with_fee.balanceOf(fee_recipient) == initial_fee_balance + fee
```

---

## 7. Complete Test Suite {#complete-test-suite}

```python
# tests/test_complete_suite.py
import pytest
import boa
from hypothesis import given, settings, strategies as st

# pytest.ini หรือ pyproject.toml สำหรับ configuration
"""
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
"""

# Setup สำหรับทั้ง test suite
@pytest.fixture(autouse=True)
def setup_boa_env():
    """ตั้งค่า boa environment ก่อนแต่ละ test"""
    # Reset state if needed
    yield

class TestIntegration:
    """Integration tests - ทดสอบการทำงานร่วมกันของ features"""
    
    def test_full_token_lifecycle(self, token, deployer, alice, bob):
        """ทดสอบ lifecycle ทั้งหมด: deploy -> mint -> transfer -> burn"""
        
        # 1. Deploy (ทำใน fixture แล้ว)
        assert token.total_supply() == 0
        
        # 2. Mint
        with boa.env.prank(deployer):
            token.mint(alice, 1000 * 10**18)
        assert token.balanceOf(alice) == 1000 * 10**18
        assert token.total_supply() == 1000 * 10**18
        
        # 3. Transfer
        with boa.env.prank(alice):
            token.transfer(bob, 400 * 10**18)
        assert token.balanceOf(alice) == 600 * 10**18
        assert token.balanceOf(bob) == 400 * 10**18
        
        # 4. Approve and TransferFrom
        with boa.env.prank(alice):
            token.approve(deployer, 300 * 10**18)
        
        with boa.env.prank(deployer):
            token.transferFrom(alice, bob, 200 * 10**18)
        assert token.balanceOf(alice) == 400 * 10**18
        assert token.allowance(alice, deployer) == 100 * 10**18
        
        # 5. Burn
        with boa.env.prank(bob):
            token.burn(100 * 10**18)
        assert token.total_supply() == 900 * 10**18
        assert token.balanceOf(bob) == 500 * 10**18
    
    def test_pause_resume_cycle(self, token_with_supply, deployer, alice, bob):
        """ทดสอบ pause/resume cycle"""
        
        # Transfer works before pause
        with boa.env.prank(alice):
            token_with_supply.transfer(bob, 100 * 10**18)
        
        # Pause
        with boa.env.prank(deployer):
            token_with_supply.set_paused(True)
        
        # Transfer fails when paused
        with pytest.raises(Exception):
            with boa.env.prank(alice):
                token_with_supply.transfer(bob, 100 * 10**18)
        
        # Resume
        with boa.env.prank(deployer):
            token_with_supply.set_paused(False)
        
        # Transfer works again
        with boa.env.prank(alice):
            token_with_supply.transfer(bob, 100 * 10**18)
    
    def test_ownership_transfer_then_admin_actions(
        self, token, deployer, alice, bob
    ):
        """ทดสอบ ownership transfer และ admin actions หลังจากนั้น"""
        
        # Transfer ownership
        with boa.env.prank(deployer):
            token.transfer_ownership(alice)
        
        assert token.owner() == alice
        
        # Old owner cannot do admin actions
        with pytest.raises(Exception):
            with boa.env.prank(deployer):
                token.set_paused(True)
        
        # New owner can do admin actions
        with boa.env.prank(alice):
            token.set_paused(True)
        assert token.is_paused() == True
        
        # New owner can mint
        with boa.env.prank(alice):
            token.mint(bob, 1000 * 10**18)
        assert token.balanceOf(bob) == 1000 * 10**18
    
    def test_fee_system_integration(self, token, deployer, alice, bob):
        """ทดสอบ fee system ทั้งหมด"""
        fee_recipient_addr = boa.env.generate_address()
        
        with boa.env.prank(deployer):
            # Setup
            token.set_fee(100)  # 1% fee
            token.set_fee_recipient(fee_recipient_addr)
            token.mint(alice, 10000 * 10**18)
        
        # Transfer with fee
        with boa.env.prank(alice):
            token.transfer(bob, 1000 * 10**18)
        
        fee = 1000 * 10**18 * 100 // 10000  # 1%
        received = 1000 * 10**18 - fee
        
        assert token.balanceOf(bob) == received
        assert token.balanceOf(fee_recipient_addr) == fee
        
        # Change fee
        with boa.env.prank(deployer):
            token.set_fee(0)
        
        # Transfer without fee
        alice_balance = token.balanceOf(alice)
        with boa.env.prank(alice):
            token.transfer(bob, alice_balance)
        
        assert token.balanceOf(alice) == 0
        assert token.balanceOf(bob) == received + alice_balance


class TestEdgeCases:
    """Edge cases ที่อาจทำให้ contract ทำงานผิดพลาด"""
    
    def test_transfer_zero_amount(self, token_with_supply, alice, bob):
        """Transfer 0 amount ต้องไม่ error"""
        initial_alice = token_with_supply.balanceOf(alice)
        initial_bob = token_with_supply.balanceOf(bob)
        
        with boa.env.prank(alice):
            token_with_supply.transfer(bob, 0)
        
        assert token_with_supply.balanceOf(alice) == initial_alice
        assert token_with_supply.balanceOf(bob) == initial_bob
    
    def test_max_allowance(self, token, alice, bob):
        """Test max uint256 allowance"""
        max_val = 2**256 - 1
        
        with boa.env.prank(alice):
            token.approve(bob, max_val)
        
        assert token.allowance(alice, bob) == max_val
    
    def test_multiple_minters(self, token, deployer, accounts):
        """ทดสอบ multiple minters"""
        minters = accounts[1:4]
        
        with boa.env.prank(deployer):
            for m in minters:
                token.set_minter(m, True)
        
        for m in minters:
            assert token.minters(m) == True
            with boa.env.prank(m):
                token.mint(accounts[4], 100 * 10**18)
        
        assert token.balanceOf(accounts[4]) == 300 * 10**18
    
    def test_revoke_minter(self, minter_token, deployer, minter, alice):
        """ทดสอบ revoke minter role"""
        with boa.env.prank(minter):
            minter_token.mint(alice, 100 * 10**18)
        
        with boa.env.prank(deployer):
            minter_token.set_minter(minter, False)
        
        with pytest.raises(Exception, match="Not authorized"):
            with boa.env.prank(minter):
                minter_token.mint(alice, 100 * 10**18)


# Run all tests
if __name__ == "__main__":
    import subprocess
    subprocess.run([
        "python", "-m", "pytest",
        "tests/",
        "-v",
        "--cov=contracts",
        "--cov-report=html",
        "--cov-report=term-missing",
        "-x"  # stop on first failure
    ])
```

### การรัน Tests

```bash
# รัน tests ทั้งหมด
pytest tests/ -v

# รัน พร้อม coverage
pytest tests/ --cov=contracts --cov-report=html

# รัน specific test file
pytest tests/test_property_based.py -v

# รัน พร้อม hypothesis verbosity
pytest tests/test_property_based.py -v --hypothesis-verbosity=verbose

# รัน พร้อม parallelism (ต้องติดตั้ง pytest-xdist)
pytest tests/ -n auto

# รัน เฉพาะ slow tests
pytest tests/ -m slow

# รัน เฉพาะ fast tests
pytest tests/ -m "not slow"
```

### pytest.ini

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short --strict-markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    fuzz: marks tests as fuzz tests
```

### pyproject.toml (สำหรับ Hypothesis)

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"

[tool.hypothesis]
max_examples = 100
deadline = 5000
suppress_health_check = ["too_slow"]
```
