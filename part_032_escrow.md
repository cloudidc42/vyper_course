# Part 032: Escrow Contract

## สารบัญ
1. [บทนำ Escrow](#บทนำ)
2. [Basic Escrow](#basic-escrow)
3. [Dispute Resolution](#dispute-resolution)
4. [Arbiter Pattern](#arbiter-pattern)
5. [Token Escrow](#token-escrow)
6. [ตัวอย่าง: Service Payment Escrow](#ตัวอย่าง-service-payment-escrow)
7. [Test Code](#test-code)

---

## บทนำ

**Escrow** คือระบบที่บุคคลที่สามที่เชื่อถือได้ (Arbiter) ถือเงินไว้แทน จนกว่าเงื่อนไขจะสำเร็จ ในโลก Smart Contract arbiter อาจเป็น contract logic เอง หรือ multi-sig

### Use Cases
- Freelance payment
- NFT marketplace
- P2P trading
- สัญญาบริการ

---

## Basic Escrow

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Basic ETH Escrow

event FundsDeposited:
    depositor: indexed(address)
    beneficiary: indexed(address)
    amount: uint256

event FundsReleased:
    beneficiary: indexed(address)
    amount: uint256

event FundsRefunded:
    depositor: indexed(address)
    amount: uint256

# States
AWAITING_PAYMENT: constant(uint8) = 0
AWAITING_DELIVERY: constant(uint8) = 1
COMPLETE: constant(uint8) = 2
REFUNDED: constant(uint8) = 3

depositor: public(address)     # ผู้จ่ายเงิน
beneficiary: public(address)   # ผู้รับเงิน
arbiter: public(address)       # ผู้ตัดสิน
amount: public(uint256)        # จำนวนเงิน
state: public(uint8)           # สถานะ

@deploy
def __init__(
    _depositor: address,
    _beneficiary: address,
    _arbiter: address
):
    self.depositor = _depositor
    self.beneficiary = _beneficiary
    self.arbiter = _arbiter
    self.state = AWAITING_PAYMENT

@external
@payable
def deposit():
    """ฝากเงินใน escrow"""
    assert msg.sender == self.depositor, "Only depositor"
    assert self.state == AWAITING_PAYMENT, "Wrong state"
    assert msg.value > 0, "Must send ETH"
    
    self.amount = msg.value
    self.state = AWAITING_DELIVERY
    
    log FundsDeposited(self.depositor, self.beneficiary, msg.value)

@external
def confirm_delivery():
    """ยืนยันการส่งมอบ -> release เงิน"""
    assert msg.sender == self.depositor, "Only depositor"
    assert self.state == AWAITING_DELIVERY, "Wrong state"
    
    self.state = COMPLETE
    
    amount_to_send: uint256 = self.amount
    self.amount = 0
    
    send(self.beneficiary, amount_to_send)
    
    log FundsReleased(self.beneficiary, amount_to_send)

@external
def refund():
    """คืนเงิน (arbiter หรือ beneficiary ยกเลิก)"""
    assert msg.sender == self.arbiter or msg.sender == self.beneficiary, \
        "Not authorized"
    assert self.state == AWAITING_DELIVERY, "Wrong state"
    
    self.state = REFUNDED
    
    amount_to_refund: uint256 = self.amount
    self.amount = 0
    
    send(self.depositor, amount_to_refund)
    
    log FundsRefunded(self.depositor, amount_to_refund)
```

---

## Dispute Resolution

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Escrow with Dispute Resolution

event Disputed:
    disputer: indexed(address)
    reason: String[200]

event DisputeResolved:
    arbiter: indexed(address)
    winner: indexed(address)
    amount: uint256

AWAITING_PAYMENT: constant(uint8) = 0
AWAITING_DELIVERY: constant(uint8) = 1
DISPUTED: constant(uint8) = 2
COMPLETE: constant(uint8) = 3
CANCELLED: constant(uint8) = 4

depositor: public(address)
beneficiary: public(address)
arbiter: public(address)
amount: public(uint256)
state: public(uint8)
dispute_deadline: public(uint256)

DISPUTE_WINDOW: constant(uint256) = 7 * 86400  # 7 วัน

@deploy
def __init__(
    _depositor: address,
    _beneficiary: address,
    _arbiter: address
):
    self.depositor = _depositor
    self.beneficiary = _beneficiary
    self.arbiter = _arbiter
    self.state = AWAITING_PAYMENT

@external
@payable
def deposit():
    assert msg.sender == self.depositor
    assert self.state == AWAITING_PAYMENT
    assert msg.value > 0
    
    self.amount = msg.value
    self.state = AWAITING_DELIVERY

@external
def confirm_delivery():
    """ยืนยัน delivery -> เริ่ม dispute window"""
    assert msg.sender == self.depositor
    assert self.state == AWAITING_DELIVERY
    
    # เริ่ม dispute window
    self.dispute_deadline = block.timestamp + DISPUTE_WINDOW
    # หมายเหตุ: ยังไม่ release ทันที รอ dispute window

@external
def raise_dispute(reason: String[200]):
    """เปิด dispute"""
    assert msg.sender == self.depositor or msg.sender == self.beneficiary
    assert self.state == AWAITING_DELIVERY
    
    self.state = DISPUTED
    log Disputed(msg.sender, reason)

@external
def resolve_dispute(favor_beneficiary: bool):
    """arbiter ตัดสิน dispute"""
    assert msg.sender == self.arbiter
    assert self.state == DISPUTED
    
    amount_to_send: uint256 = self.amount
    self.amount = 0
    
    if favor_beneficiary:
        self.state = COMPLETE
        send(self.beneficiary, amount_to_send)
        log DisputeResolved(self.arbiter, self.beneficiary, amount_to_send)
    else:
        self.state = CANCELLED
        send(self.depositor, amount_to_send)
        log DisputeResolved(self.arbiter, self.depositor, amount_to_send)

@external
def claim_after_window():
    """Beneficiary รับเงินหลัง dispute window"""
    assert msg.sender == self.beneficiary
    assert self.state == AWAITING_DELIVERY
    assert self.dispute_deadline > 0
    assert block.timestamp > self.dispute_deadline
    
    self.state = COMPLETE
    amount_to_send: uint256 = self.amount
    self.amount = 0
    send(self.beneficiary, amount_to_send)
```

---

## Arbiter Pattern

### Multi-Arbiter System

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Multi-Arbiter Escrow

MAX_ARBITERS: constant(uint256) = 5

struct EscrowDeal:
    depositor: address
    beneficiary: address
    amount: uint256
    state: uint8
    arbiter_votes_release: uint256
    arbiter_votes_refund: uint256
    created_at: uint256
    deadline: uint256

OPEN: constant(uint8) = 0
FUNDED: constant(uint8) = 1
DISPUTED: constant(uint8) = 2
RELEASED: constant(uint8) = 3
REFUNDED: constant(uint8) = 4

arbiters: public(DynArray[address, MAX_ARBITERS])
is_arbiter: public(HashMap[address, bool])
required_arbiter_votes: public(uint256)

deal_count: public(uint256)
deals: public(HashMap[uint256, EscrowDeal])
arbiter_voted: HashMap[uint256, HashMap[address, bool]]

@deploy
def __init__(
    _arbiters: DynArray[address, MAX_ARBITERS],
    _required_votes: uint256
):
    assert len(_arbiters) >= _required_votes
    
    for a: address in _arbiters:
        self.is_arbiter[a] = True
        self.arbiters.append(a)
    
    self.required_arbiter_votes = _required_votes

@external
def create_deal(
    beneficiary: address,
    deadline: uint256
) -> uint256:
    """สร้าง escrow deal ใหม่"""
    assert beneficiary != empty(address)
    assert deadline > block.timestamp
    
    deal_id: uint256 = self.deal_count
    self.deals[deal_id] = EscrowDeal({
        depositor: msg.sender,
        beneficiary: beneficiary,
        amount: 0,
        state: OPEN,
        arbiter_votes_release: 0,
        arbiter_votes_refund: 0,
        created_at: block.timestamp,
        deadline: deadline
    })
    
    self.deal_count += 1
    return deal_id

@external
@payable
def fund_deal(deal_id: uint256):
    """ฝากเงินใน deal"""
    deal: EscrowDeal = self.deals[deal_id]
    assert msg.sender == deal.depositor
    assert deal.state == OPEN
    assert msg.value > 0
    
    self.deals[deal_id].amount = msg.value
    self.deals[deal_id].state = FUNDED

@external
def vote_release(deal_id: uint256):
    """Arbiter vote release เงิน"""
    assert self.is_arbiter[msg.sender]
    assert not self.arbiter_voted[deal_id][msg.sender]
    
    deal: EscrowDeal = self.deals[deal_id]
    assert deal.state == FUNDED or deal.state == DISPUTED
    
    self.arbiter_voted[deal_id][msg.sender] = True
    self.deals[deal_id].arbiter_votes_release += 1
    
    if self.deals[deal_id].arbiter_votes_release >= self.required_arbiter_votes:
        self.deals[deal_id].state = RELEASED
        amount: uint256 = deal.amount
        self.deals[deal_id].amount = 0
        send(deal.beneficiary, amount)

@external
def vote_refund(deal_id: uint256):
    """Arbiter vote refund"""
    assert self.is_arbiter[msg.sender]
    assert not self.arbiter_voted[deal_id][msg.sender]
    
    deal: EscrowDeal = self.deals[deal_id]
    assert deal.state == FUNDED or deal.state == DISPUTED
    
    self.arbiter_voted[deal_id][msg.sender] = True
    self.deals[deal_id].arbiter_votes_refund += 1
    
    if self.deals[deal_id].arbiter_votes_refund >= self.required_arbiter_votes:
        self.deals[deal_id].state = REFUNDED
        amount: uint256 = deal.amount
        self.deals[deal_id].amount = 0
        send(deal.depositor, amount)
```

---

## Token Escrow

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token Escrow

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

event TokenDeposited:
    token: indexed(address)
    depositor: indexed(address)
    amount: uint256

event TokenReleased:
    token: indexed(address)
    beneficiary: indexed(address)
    amount: uint256

struct TokenEscrow:
    token: address
    depositor: address
    beneficiary: address
    arbiter: address
    amount: uint256
    released: bool
    refunded: bool
    deadline: uint256

escrow_count: public(uint256)
escrows: public(HashMap[uint256, TokenEscrow])

@deploy
def __init__():
    pass

@external
def create_token_escrow(
    token: address,
    beneficiary: address,
    arbiter: address,
    amount: uint256,
    duration: uint256
) -> uint256:
    """สร้าง token escrow"""
    assert token != empty(address)
    assert beneficiary != empty(address)
    assert amount > 0
    
    escrow_id: uint256 = self.escrow_count
    
    # โอน token เข้า contract
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    self.escrows[escrow_id] = TokenEscrow({
        token: token,
        depositor: msg.sender,
        beneficiary: beneficiary,
        arbiter: arbiter,
        amount: amount,
        released: False,
        refunded: False,
        deadline: block.timestamp + duration
    })
    
    self.escrow_count += 1
    
    log TokenDeposited(token, msg.sender, amount)
    return escrow_id

@external
def release_tokens(escrow_id: uint256):
    """Release tokens ไปยัง beneficiary"""
    escrow: TokenEscrow = self.escrows[escrow_id]
    assert msg.sender == escrow.depositor or msg.sender == escrow.arbiter
    assert not escrow.released and not escrow.refunded
    
    self.escrows[escrow_id].released = True
    
    ERC20(escrow.token).transfer(escrow.beneficiary, escrow.amount)
    
    log TokenReleased(escrow.token, escrow.beneficiary, escrow.amount)

@external
def refund_tokens(escrow_id: uint256):
    """Refund tokens ให้ depositor"""
    escrow: TokenEscrow = self.escrows[escrow_id]
    assert msg.sender == escrow.arbiter or (
        msg.sender == escrow.depositor and block.timestamp > escrow.deadline
    )
    assert not escrow.released and not escrow.refunded
    
    self.escrows[escrow_id].refunded = True
    
    ERC20(escrow.token).transfer(escrow.depositor, escrow.amount)
```

---

## ตัวอย่าง: Service Payment Escrow

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Service Payment Escrow
# @notice ระบบ Escrow สำหรับการจ้างงาน Service

# ==================== Enums/Constants ====================

SERVICE_PENDING: constant(uint8) = 0
SERVICE_STARTED: constant(uint8) = 1
SERVICE_DELIVERED: constant(uint8) = 2
PAYMENT_RELEASED: constant(uint8) = 3
DISPUTED: constant(uint8) = 4
CANCELLED: constant(uint8) = 5

# ==================== Events ====================

event ServiceCreated:
    service_id: indexed(uint256)
    client: indexed(address)
    provider: indexed(address)
    amount: uint256

event ServiceStarted:
    service_id: indexed(uint256)

event ServiceDelivered:
    service_id: indexed(uint256)

event PaymentReleased:
    service_id: indexed(uint256)
    amount: uint256

event DisputeOpened:
    service_id: indexed(uint256)
    opener: indexed(address)

event DisputeResolved:
    service_id: indexed(uint256)
    client_amount: uint256
    provider_amount: uint256

event ReviewSubmitted:
    service_id: indexed(uint256)
    reviewer: indexed(address)
    rating: uint8

# ==================== Structs ====================

struct ServiceAgreement:
    client: address
    provider: address
    arbiter: address
    amount: uint256
    platform_fee_bps: uint256
    state: uint8
    deadline: uint256
    dispute_deadline: uint256
    created_at: uint256
    description: String[500]

struct ServiceReview:
    rating: uint8         # 1-5
    comment: String[500]
    timestamp: uint256
    from_client: bool

# ==================== State Variables ====================

owner: public(address)
platform_fee_bps: public(uint256)  # Platform fee (100 = 1%)
fee_collector: public(address)
default_arbiter: public(address)

service_count: public(uint256)
services: public(HashMap[uint256, ServiceAgreement])
reviews: public(HashMap[uint256, ServiceReview])

DISPUTE_WINDOW: constant(uint256) = 3 * 86400   # 3 วัน

# ==================== Constructor ====================

@deploy
def __init__(
    _fee_bps: uint256,
    _fee_collector: address,
    _default_arbiter: address
):
    self.owner = msg.sender
    self.platform_fee_bps = _fee_bps
    self.fee_collector = _fee_collector
    self.default_arbiter = _default_arbiter

# ==================== Create Service ====================

@external
@payable
def create_service(
    provider: address,
    description: String[500],
    duration_days: uint256,
    custom_arbiter: address
) -> uint256:
    """
    @notice Client สร้าง service agreement พร้อมฝากเงิน
    @return service_id
    """
    assert provider != empty(address), "Invalid provider"
    assert provider != msg.sender, "Client cannot be provider"
    assert msg.value > 0, "Must send payment"
    assert duration_days > 0 and duration_days <= 365
    
    arbiter: address = custom_arbiter
    if arbiter == empty(address):
        arbiter = self.default_arbiter
    
    service_id: uint256 = self.service_count
    
    self.services[service_id] = ServiceAgreement({
        client: msg.sender,
        provider: provider,
        arbiter: arbiter,
        amount: msg.value,
        platform_fee_bps: self.platform_fee_bps,
        state: SERVICE_PENDING,
        deadline: block.timestamp + duration_days * 86400,
        dispute_deadline: 0,
        created_at: block.timestamp,
        description: description
    })
    
    self.service_count += 1
    
    log ServiceCreated(service_id, msg.sender, provider, msg.value)
    return service_id

# ==================== Service Flow ====================

@external
def start_service(service_id: uint256):
    """Provider ยืนยันรับงาน"""
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.provider, "Not provider"
    assert service.state == SERVICE_PENDING, "Wrong state"
    assert block.timestamp <= service.deadline, "Deadline passed"
    
    self.services[service_id].state = SERVICE_STARTED
    log ServiceStarted(service_id)

@external
def mark_delivered(service_id: uint256):
    """Provider แจ้ง delivery"""
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.provider, "Not provider"
    assert service.state == SERVICE_STARTED, "Service not started"
    
    self.services[service_id].state = SERVICE_DELIVERED
    self.services[service_id].dispute_deadline = block.timestamp + DISPUTE_WINDOW
    
    log ServiceDelivered(service_id)

@external
def approve_delivery(service_id: uint256):
    """
    @notice Client อนุมัติ delivery -> release payment
    """
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.client, "Not client"
    assert service.state == SERVICE_DELIVERED, "Not delivered"
    
    self._release_payment(service_id)

@external
def auto_release(service_id: uint256):
    """
    @notice Auto-release หลัง dispute window ผ่าน
    """
    service: ServiceAgreement = self.services[service_id]
    assert service.state == SERVICE_DELIVERED, "Not delivered"
    assert block.timestamp > service.dispute_deadline, "Dispute window not expired"
    
    self._release_payment(service_id)

@internal
def _release_payment(service_id: uint256):
    """Internal: release payment ให้ provider"""
    service: ServiceAgreement = self.services[service_id]
    
    self.services[service_id].state = PAYMENT_RELEASED
    
    # คำนวณ fee
    platform_fee: uint256 = service.amount * service.platform_fee_bps / 10000
    provider_amount: uint256 = service.amount - platform_fee
    
    # Send platform fee
    if platform_fee > 0:
        send(self.fee_collector, platform_fee)
    
    # Send to provider
    send(service.provider, provider_amount)
    
    log PaymentReleased(service_id, provider_amount)

# ==================== Dispute Functions ====================

@external
def open_dispute(service_id: uint256):
    """Client เปิด dispute ภายใน dispute window"""
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.client, "Not client"
    assert service.state == SERVICE_DELIVERED, "Not delivered"
    assert block.timestamp <= service.dispute_deadline, "Dispute window closed"
    
    self.services[service_id].state = DISPUTED
    log DisputeOpened(service_id, msg.sender)

@external
def resolve_dispute(
    service_id: uint256,
    client_percentage: uint256  # 0-100
):
    """
    @notice Arbiter ตัดสิน dispute
    @param client_percentage เปอร์เซ็นต์ที่ client ได้ (0-100)
    """
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.arbiter, "Not arbiter"
    assert service.state == DISPUTED, "Not disputed"
    assert client_percentage <= 100, "Invalid percentage"
    
    self.services[service_id].state = CANCELLED
    
    # คำนวณจำนวน
    total: uint256 = service.amount
    platform_fee: uint256 = total * service.platform_fee_bps / 10000
    distributable: uint256 = total - platform_fee
    
    client_amount: uint256 = distributable * client_percentage / 100
    provider_amount: uint256 = distributable - client_amount
    
    # ส่ง platform fee
    if platform_fee > 0:
        send(self.fee_collector, platform_fee)
    
    # ส่งให้ client
    if client_amount > 0:
        send(service.client, client_amount)
    
    # ส่งให้ provider
    if provider_amount > 0:
        send(service.provider, provider_amount)
    
    log DisputeResolved(service_id, client_amount, provider_amount)

@external
def cancel_service(service_id: uint256):
    """
    @notice ยกเลิก service (client หรือ provider ก่อน start)
    """
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.client or msg.sender == service.provider
    assert service.state == SERVICE_PENDING, "Cannot cancel after start"
    
    self.services[service_id].state = CANCELLED
    
    # Refund client
    send(service.client, service.amount)

@external
def refund_expired(service_id: uint256):
    """Refund ถ้า provider ไม่ start ภายใน deadline"""
    service: ServiceAgreement = self.services[service_id]
    assert msg.sender == service.client
    assert service.state == SERVICE_PENDING
    assert block.timestamp > service.deadline
    
    self.services[service_id].state = CANCELLED
    send(service.client, service.amount)

# ==================== Review System ====================

@external
def submit_review(
    service_id: uint256,
    rating: uint8,
    comment: String[500]
):
    """
    @notice ส่ง review หลัง service เสร็จ
    """
    service: ServiceAgreement = self.services[service_id]
    assert service.state == PAYMENT_RELEASED, "Service not complete"
    assert msg.sender == service.client or msg.sender == service.provider
    assert rating >= 1 and rating <= 5, "Invalid rating"
    assert self.reviews[service_id].timestamp == 0, "Already reviewed"
    
    from_client: bool = msg.sender == service.client
    
    self.reviews[service_id] = ServiceReview({
        rating: rating,
        comment: comment,
        timestamp: block.timestamp,
        from_client: from_client
    })
    
    log ReviewSubmitted(service_id, msg.sender, rating)

# ==================== Admin Functions ====================

@external
def update_platform_fee(new_fee_bps: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert new_fee_bps <= 500, "Max 5% fee"  # สูงสุด 5%
    self.platform_fee_bps = new_fee_bps

@external
def update_default_arbiter(new_arbiter: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_arbiter != empty(address)
    self.default_arbiter = new_arbiter

# ==================== View Functions ====================

@view
@external
def get_service(service_id: uint256) -> ServiceAgreement:
    return self.services[service_id]

@view
@external
def get_review(service_id: uint256) -> ServiceReview:
    return self.reviews[service_id]

@view
@external
def is_disputable(service_id: uint256) -> bool:
    service: ServiceAgreement = self.services[service_id]
    return (service.state == SERVICE_DELIVERED and
            block.timestamp <= service.dispute_deadline)

@view
@external
def can_auto_release(service_id: uint256) -> bool:
    service: ServiceAgreement = self.services[service_id]
    return (service.state == SERVICE_DELIVERED and
            block.timestamp > service.dispute_deadline)
```

---

## Test Code

```python
# tests/test_escrow.py
import pytest

ONE_DAY = 86400

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def client(accounts):
    return accounts[1]

@pytest.fixture
def provider(accounts):
    return accounts[2]

@pytest.fixture
def arbiter(accounts):
    return accounts[3]

@pytest.fixture
def fee_collector(accounts):
    return accounts[4]

@pytest.fixture
def escrow(owner, arbiter, fee_collector, project):
    return project.ServiceEscrow.deploy(
        100,  # 1% fee
        fee_collector.address,
        arbiter.address,
        sender=owner
    )

class TestEscrowCreation:
    
    def test_create_service(self, escrow, client, provider):
        """ทดสอบสร้าง service"""
        amount = 1 * 10**18
        service_id = escrow.create_service(
            provider.address,
            "Build a website",
            30,
            "0x0000000000000000000000000000000000000000",
            sender=client,
            value=amount
        )
        
        service = escrow.get_service(service_id)
        assert service.client == client.address
        assert service.provider == provider.address
        assert service.amount == amount
    
    def test_client_cannot_be_provider(self, escrow, client):
        """Client ไม่สามารถเป็น provider เองได้"""
        with pytest.raises(Exception):
            escrow.create_service(
                client.address,
                "Test",
                30,
                "0x0000000000000000000000000000000000000000",
                sender=client,
                value=10**18
            )

class TestServiceFlow:
    
    def test_full_happy_path(self, escrow, client, provider, fee_collector):
        """ทดสอบ flow ปกติ"""
        amount = 10 * 10**18
        
        # Create
        service_id = escrow.create_service(
            provider.address,
            "Build website",
            30,
            "0x0000000000000000000000000000000000000000",
            sender=client,
            value=amount
        )
        
        # Start
        escrow.start_service(service_id, sender=provider)
        
        # Mark delivered
        escrow.mark_delivered(service_id, sender=provider)
        
        # Approve
        before_provider = provider.balance
        before_fee = fee_collector.balance
        
        escrow.approve_delivery(service_id, sender=client)
        
        # ตรวจสอบ fee
        expected_fee = amount * 100 // 10000  # 1%
        assert fee_collector.balance - before_fee == expected_fee
        assert provider.balance - before_provider == amount - expected_fee
    
    def test_dispute_resolved_for_client(self, escrow, client, provider, arbiter):
        """ทดสอบ dispute ที่ client ชนะ"""
        amount = 10 * 10**18
        
        service_id = escrow.create_service(
            provider.address, "Service", 30,
            "0x0000000000000000000000000000000000000000",
            sender=client, value=amount
        )
        
        escrow.start_service(service_id, sender=provider)
        escrow.mark_delivered(service_id, sender=provider)
        escrow.open_dispute(service_id, sender=client)
        
        before_client = client.balance
        
        # Arbiter returns 80% to client
        escrow.resolve_dispute(service_id, 80, sender=arbiter)
        
        # คำนวณ expected
        fee = amount * 100 // 10000  # 1%
        distributable = amount - fee
        client_amount = distributable * 80 // 100
        
        assert client.balance - before_client == client_amount
    
    def test_cancel_before_start(self, escrow, client, provider):
        """ทดสอบ cancel ก่อน start"""
        amount = 10**18
        
        service_id = escrow.create_service(
            provider.address, "Service", 30,
            "0x0000000000000000000000000000000000000000",
            sender=client, value=amount
        )
        
        before = client.balance
        escrow.cancel_service(service_id, sender=client)
        assert client.balance > before  # ได้รับเงินคืน
```

---

## สรุป

Escrow Pattern เป็น Pattern สำคัญสำหรับ trustless transactions:

| Feature | Description |
|---------|-------------|
| Basic Escrow | Simple release/refund |
| Dispute Resolution | Arbiter ตัดสิน |
| Auto-release | Release โดยอัตโนมัติหลัง window |
| Token Escrow | รองรับ ERC20 tokens |

---

[⬅️ Part 031: Multisig Wallet](part_031_multisig.md) | [Part 033: Voting System ➡️](part_033_voting.md)
