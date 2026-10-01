# Part 011: Structs

## สารบัญ
1. [Struct Declaration](#declaration)
2. [Struct ใน Storage](#storage)
3. [Struct เป็น Function Parameter](#func-param)
4. [Nested Structs](#nested)
5. [Array of Structs](#array-structs)
6. [Struct Patterns](#patterns)
7. [ตัวอย่าง: NFT Metadata Contract](#example)

---

## 1. Struct Declaration {#declaration}

Struct เป็นการรวมหลาย types เข้าด้วยกันเป็นหน่วยเดียว

```python
# @version 0.4.0

# ════════════════════════════════════════
# Struct Declaration
# ════════════════════════════════════════

struct BasicInfo:
    name: String[50]
    age: uint256
    active: bool

struct Token:
    name: String[30]
    symbol: String[10]
    decimals: uint8
    total_supply: uint256

struct Position:
    owner: address
    amount: uint256
    entry_price: uint256
    entry_time: uint256
    is_open: bool

# Struct ที่มีทุก primitive types
struct FullExample:
    a: uint256
    b: int256
    c: uint8
    d: int128
    e: bool
    f: address
    g: bytes32
    h: String[100]
    i: Bytes[64]
```

### สร้าง Struct Instance

```python
# @version 0.4.0

struct Point:
    x: uint256
    y: uint256

struct Rectangle:
    top_left: uint256
    top_right: uint256
    width: uint256
    height: uint256

@pure
@external
def create_point(x: uint256, y: uint256) -> Point:
    # Method 1: Named fields
    p: Point = Point({x: x, y: y})
    return p

@pure
@external
def create_rect(x: uint256, y: uint256, w: uint256, h: uint256) -> Rectangle:
    r: Rectangle = Rectangle({
        top_left: x,
        top_right: x + w,
        width: w,
        height: h
    })
    return r

@pure
@external
def origin() -> Point:
    # Empty struct = default values
    return empty(Point)

@pure
@external
def move_point(p: Point, dx: uint256, dy: uint256) -> Point:
    return Point({x: p.x + dx, y: p.y + dy})
```

### เข้าถึง Struct Fields

```python
# @version 0.4.0

struct Player:
    name: String[30]
    score: uint256
    level: uint8
    wins: uint256
    losses: uint256

players: HashMap[address, Player]

@deploy
def __init__():
    pass

@external
def create_player(name: String[30]):
    self.players[msg.sender] = Player({
        name: name,
        score: 0,
        level: 1,
        wins: 0,
        losses: 0
    })

@external
def win_game(points: uint256):
    # Access and modify individual fields
    self.players[msg.sender].score += points
    self.players[msg.sender].wins += 1

    # Level up if score threshold reached
    if self.players[msg.sender].score >= 1000 * self.players[msg.sender].level:
        self.players[msg.sender].level += 1

@view
@external
def get_player_name(addr: address) -> String[30]:
    return self.players[addr].name

@view
@external
def get_win_rate(addr: address) -> uint256:
    p: Player = self.players[addr]
    total: uint256 = p.wins + p.losses
    if total == 0:
        return 0
    return (p.wins * 10000) / total  # Basis points
```

---

## 2. Struct ใน Storage {#storage}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Structs in Storage
# ════════════════════════════════════════

struct Config:
    min_deposit: uint256
    max_deposit: uint256
    fee_bps: uint256
    paused: bool
    owner: address
    last_updated: uint256

# Single struct in storage
config: Config

# Mapping of structs
struct UserAccount:
    balance: uint256
    locked_until: uint256
    tier: uint8
    referrer: address
    total_deposited: uint256

accounts: HashMap[address, UserAccount]

# Array of structs
struct Transaction:
    sender: address
    recipient: address
    amount: uint256
    timestamp: uint256

history: DynArray[Transaction, 10000]

@deploy
def __init__(
    min_dep: uint256,
    max_dep: uint256,
    fee: uint256
):
    self.config = Config({
        min_deposit: min_dep,
        max_deposit: max_dep,
        fee_bps: fee,
        paused: False,
        owner: msg.sender,
        last_updated: block.timestamp
    })

@external
def update_config(min_dep: uint256, max_dep: uint256):
    assert msg.sender == self.config.owner, "Not owner"
    assert min_dep < max_dep, "Invalid range"
    self.config.min_deposit = min_dep
    self.config.max_deposit = max_dep
    self.config.last_updated = block.timestamp

@payable
@external
def deposit(referrer: address):
    assert not self.config.paused, "Paused"
    assert msg.value >= self.config.min_deposit, "Too small"
    assert msg.value <= self.config.max_deposit, "Too large"

    # Update or create account
    if self.accounts[msg.sender].total_deposited == 0 and referrer != empty(address):
        self.accounts[msg.sender].referrer = referrer

    self.accounts[msg.sender].balance += msg.value
    self.accounts[msg.sender].total_deposited += msg.value

    # Record transaction
    self.history.append(Transaction({
        sender: msg.sender,
        recipient: self,
        amount: msg.value,
        timestamp: block.timestamp
    }))
```

### Storage Layout ของ Struct

```python
# @version 0.4.0

# Storage slots ของ Struct จัดเรียงตาม field order
struct Example:
    a: uint256    # slot 0
    b: address    # slot 1 (20 bytes, packed with bool)
    c: bool       # slot 1 (1 byte)
    d: uint256    # slot 2

# Vyper จัด pack primitive types เล็กๆ ในรุ่นใหม่

example: Example

@deploy
def __init__():
    self.example = Example({
        a: 42,
        b: msg.sender,
        c: True,
        d: 100
    })
```

---

## 3. Struct เป็น Function Parameter {#func-param}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Structs as Function Parameters/Returns
# ════════════════════════════════════════

struct Order:
    buyer: address
    seller: address
    item_id: uint256
    quantity: uint256
    price_per_unit: uint256
    expires_at: uint256
    is_filled: bool

struct OrderResult:
    order_id: bytes32
    total_price: uint256
    fee: uint256
    net_to_seller: uint256

orders: HashMap[bytes32, Order]
FEE_BPS: constant(uint256) = 250  # 2.5%

@deploy
def __init__():
    pass

# Struct ใน parameter
@external
def place_order(order: Order) -> bytes32:
    assert order.buyer == msg.sender, "Not buyer"
    assert order.quantity > 0, "Zero quantity"
    assert order.price_per_unit > 0, "Zero price"
    assert order.expires_at > block.timestamp, "Expired"

    order_id: bytes32 = keccak256(
        concat(
            convert(order.buyer, bytes32),
            convert(order.item_id, bytes32),
            convert(block.timestamp, bytes32)
        )
    )
    self.orders[order_id] = order
    return order_id

# Struct เป็น return value
@view
@external
def calculate_order(order_id: bytes32) -> OrderResult:
    order: Order = self.orders[order_id]
    assert order.buyer != empty(address), "Order not found"

    total: uint256 = order.price_per_unit * order.quantity
    fee: uint256 = (total * FEE_BPS) / 10000
    net: uint256 = total - fee

    return OrderResult({
        order_id: order_id,
        total_price: total,
        fee: fee,
        net_to_seller: net
    })

# Multiple struct operations
@pure
@external
def merge_orders(
    o1: Order,
    o2: Order
) -> (uint256, uint256):
    """Calculate combined order value"""
    total1: uint256 = o1.price_per_unit * o1.quantity
    total2: uint256 = o2.price_per_unit * o2.quantity
    return total1, total2
```

### Memory vs Storage Struct

```python
# @version 0.4.0

struct Data:
    value: uint256
    label: String[20]
    active: bool

stored_data: Data

@deploy
def __init__():
    self.stored_data = Data({value: 100, label: "init", active: True})

@view
@external
def demo_memory_copy() -> uint256:
    # Memory copy (ไม่กระทบ storage)
    local_copy: Data = self.stored_data
    local_copy.value = 999  # ไม่เปลี่ยน storage!
    return self.stored_data.value  # ยังเป็น 100

@external
def update_value(new_val: uint256):
    # ต้องเขียนกลับ storage
    self.stored_data.value = new_val  # โดยตรง

@external
def update_via_copy(new_val: uint256):
    # Copy, modify, write back
    copy: Data = self.stored_data
    copy.value = new_val
    copy.label = "updated"
    self.stored_data = copy  # Write back ทั้ง struct
```

---

## 4. Nested Structs {#nested}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Nested Structs
# ════════════════════════════════════════

struct Address:
    street: String[100]
    city: String[50]
    country: String[30]
    postal_code: String[10]

struct ContactInfo:
    email_hash: bytes32
    phone_hash: bytes32
    mailing_address: Address

struct UserProfile:
    name: String[50]
    joined_at: uint256
    contact: ContactInfo
    reputation: uint256

profiles: HashMap[address, UserProfile]

@external
def create_profile(
    name: String[50],
    email_h: bytes32,
    city: String[50],
    country: String[30]
):
    assert len(name) > 0, "Name required"
    self.profiles[msg.sender] = UserProfile({
        name: name,
        joined_at: block.timestamp,
        contact: ContactInfo({
            email_hash: email_h,
            phone_hash: empty(bytes32),
            mailing_address: Address({
                street: "",
                city: city,
                country: country,
                postal_code: ""
            })
        }),
        reputation: 0
    })

@view
@external
def get_city(user: address) -> String[50]:
    return self.profiles[user].contact.mailing_address.city

@external
def update_phone(phone_hash: bytes32):
    self.profiles[msg.sender].contact.phone_hash = phone_hash

@external
def add_reputation(user: address, points: uint256):
    self.profiles[user].reputation += points

@view
@external
def get_profile(user: address) -> UserProfile:
    return self.profiles[user]
```

### Nested Struct Update Pattern

```python
# @version 0.4.0

struct InnerData:
    x: uint256
    y: uint256

struct OuterData:
    id: uint256
    inner: InnerData
    label: String[20]

data_map: HashMap[uint256, OuterData]

@external
def update_inner(id: uint256, new_x: uint256):
    # Direct field access is most gas-efficient
    self.data_map[id].inner.x = new_x

@external
def update_inner_both(id: uint256, x: uint256, y: uint256):
    # Update multiple inner fields
    self.data_map[id].inner.x = x
    self.data_map[id].inner.y = y

@external
def replace_inner(id: uint256, new_inner: InnerData):
    # Replace entire inner struct
    self.data_map[id].inner = new_inner
```

---

## 5. Array of Structs {#array-structs}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Array of Structs
# ════════════════════════════════════════

struct Bid:
    bidder: address
    amount: uint256
    timestamp: uint256
    message: String[100]

struct Auction:
    item_id: uint256
    seller: address
    start_price: uint256
    end_time: uint256
    highest_bid: uint256
    highest_bidder: address
    is_settled: bool

# DynArray of structs
bids: DynArray[Bid, 1000]

# Fixed array of structs
recent_auctions: Auction[10]

# Mapping to DynArray of structs
user_bids: HashMap[address, DynArray[Bid, 100]]

auctions: HashMap[uint256, Auction]

@deploy
def __init__():
    pass

@external
def create_auction(
    item_id: uint256,
    start_price: uint256,
    duration: uint256
):
    assert self.auctions[item_id].seller == empty(address), "Auction exists"
    self.auctions[item_id] = Auction({
        item_id: item_id,
        seller: msg.sender,
        start_price: start_price,
        end_time: block.timestamp + duration,
        highest_bid: 0,
        highest_bidder: empty(address),
        is_settled: False
    })

@payable
@external
def place_bid(item_id: uint256, message: String[100]):
    auction: Auction = self.auctions[item_id]
    assert auction.seller != empty(address), "No auction"
    assert block.timestamp < auction.end_time, "Ended"
    assert msg.value > auction.highest_bid, "Bid too low"
    assert msg.value >= auction.start_price, "Below start price"

    # Refund previous highest bidder
    if auction.highest_bidder != empty(address):
        send(auction.highest_bidder, auction.highest_bid)

    # Update auction
    self.auctions[item_id].highest_bid = msg.value
    self.auctions[item_id].highest_bidder = msg.sender

    # Record bid
    new_bid: Bid = Bid({
        bidder: msg.sender,
        amount: msg.value,
        timestamp: block.timestamp,
        message: message
    })
    self.bids.append(new_bid)
    self.user_bids[msg.sender].append(new_bid)

@view
@external
def get_top_bids(item_id: uint256, n: uint256) -> DynArray[Bid, 10]:
    assert n <= 10, "Max 10"
    result: DynArray[Bid, 10] = []
    # Get last n bids for this item
    count: uint256 = 0
    total: uint256 = convert(len(self.bids), uint256)
    for i: uint256 in range(total, total, bound=1000):
        if count >= n:
            break
        # Iterate in reverse
        idx: uint256 = total - 1 - i
        if self.bids[idx].bidder != empty(address):
            result.append(self.bids[idx])
            count += 1
    return result

@view
@external
def get_user_bid_count(user: address) -> uint256:
    return convert(len(self.user_bids[user]), uint256)

@view
@external
def get_auction(item_id: uint256) -> Auction:
    return self.auctions[item_id]
```

---

## 6. Struct Patterns {#patterns}

### Pattern: State Machine

```python
# @version 0.4.0

# States
STATE_CREATED: constant(uint8) = 0
STATE_ACTIVE: constant(uint8) = 1
STATE_PAUSED: constant(uint8) = 2
STATE_COMPLETED: constant(uint8) = 3
STATE_CANCELLED: constant(uint8) = 4

struct Campaign:
    id: uint256
    creator: address
    goal: uint256
    raised: uint256
    deadline: uint256
    state: uint8
    title: String[100]

campaigns: HashMap[uint256, Campaign]
next_id: uint256

@deploy
def __init__():
    self.next_id = 1

@external
def create_campaign(
    goal: uint256,
    duration: uint256,
    title: String[100]
) -> uint256:
    id: uint256 = self.next_id
    self.campaigns[id] = Campaign({
        id: id,
        creator: msg.sender,
        goal: goal,
        raised: 0,
        deadline: block.timestamp + duration,
        state: STATE_CREATED,
        title: title
    })
    self.next_id += 1
    return id

@external
def activate(id: uint256):
    assert self.campaigns[id].creator == msg.sender, "Not creator"
    assert self.campaigns[id].state == STATE_CREATED, "Invalid state"
    self.campaigns[id].state = STATE_ACTIVE

@payable
@external
def donate(id: uint256):
    assert self.campaigns[id].state == STATE_ACTIVE, "Not active"
    assert block.timestamp < self.campaigns[id].deadline, "Ended"
    self.campaigns[id].raised += msg.value

@external
def complete(id: uint256):
    campaign: Campaign = self.campaigns[id]
    assert campaign.state == STATE_ACTIVE, "Not active"
    assert campaign.raised >= campaign.goal, "Goal not reached"
    self.campaigns[id].state = STATE_COMPLETED
    send(campaign.creator, campaign.raised)
```

### Pattern: Config/Settings Struct

```python
# @version 0.4.0

struct ProtocolConfig:
    owner: address
    treasury: address
    fee_bps: uint256
    min_amount: uint256
    max_amount: uint256
    paused: bool
    version: uint8

config: ProtocolConfig
pending_config: ProtocolConfig
config_change_time: uint256
CONFIG_DELAY: constant(uint256) = 2 * 24 * 3600  # 2 days

@deploy
def __init__(treasury: address, fee: uint256):
    self.config = ProtocolConfig({
        owner: msg.sender,
        treasury: treasury,
        fee_bps: fee,
        min_amount: 10**15,
        max_amount: 10**22,
        paused: False,
        version: 1
    })

@external
def propose_config_change(new_config: ProtocolConfig):
    assert msg.sender == self.config.owner, "Not owner"
    self.pending_config = new_config
    self.config_change_time = block.timestamp + CONFIG_DELAY

@external
def apply_config_change():
    assert msg.sender == self.config.owner, "Not owner"
    assert block.timestamp >= self.config_change_time, "Too early"
    assert self.config_change_time > 0, "No pending change"
    self.config = self.pending_config
    self.config_change_time = 0
```

---

## 7. ตัวอย่าง: NFT Metadata Contract {#example}

Contract จัดการ NFT Metadata ครบสมบูรณ์

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# NFTCollection Contract
# จัดการ NFT พร้อม on-chain metadata ครบถ้วน
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━

struct Attribute:
    trait_type: String[30]
    value: String[50]

struct NFTMetadata:
    name: String[100]
    description: String[500]
    image_uri: String[200]
    external_url: String[200]
    background_color: String[10]  # hex color, e.g. "FF0000"
    animation_url: String[200]

struct NFTToken:
    token_id: uint256
    owner: address
    metadata: NFTMetadata
    minted_at: uint256
    last_transferred: uint256
    transfer_count: uint256
    is_locked: bool    # Soul-bound if True

struct CollectionInfo:
    name: String[100]
    symbol: String[10]
    max_supply: uint256
    minted_count: uint256
    base_uri: String[200]
    creator: address
    royalty_bps: uint256
    royalty_receiver: address

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
MAX_SUPPLY: constant(uint256) = 10000
MAX_ATTRIBUTES: constant(uint256) = 20

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
collection: CollectionInfo

# Token storage
tokens: HashMap[uint256, NFTToken]
attributes: HashMap[uint256, DynArray[Attribute, 20]]  # token_id -> attributes

# Ownership
token_owner: HashMap[uint256, address]
owned_tokens: HashMap[address, DynArray[uint256, 1000]]  # owner -> token_ids
owned_tokens_index: HashMap[uint256, uint256]  # token_id -> index in owned_tokens

# Approvals
token_approvals: HashMap[uint256, address]
operator_approvals: HashMap[address, HashMap[address, bool]]

# Minting
minter: address
mint_price: uint256
is_public_mint: bool

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    token_id: indexed(uint256)

event Approval:
    owner: indexed(address)
    approved: indexed(address)
    token_id: indexed(uint256)

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

event MetadataUpdate:
    token_id: indexed(uint256)

event Minted:
    to: indexed(address)
    token_id: indexed(uint256)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__(
    name: String[100],
    symbol: String[10],
    max_supply: uint256,
    royalty_bps: uint256,
    royalty_receiver: address
):
    self.owner = msg.sender
    self.minter = msg.sender
    self.collection = CollectionInfo({
        name: name,
        symbol: symbol,
        max_supply: max_supply,
        minted_count: 0,
        base_uri: "",
        creator: msg.sender,
        royalty_bps: royalty_bps,
        royalty_receiver: royalty_receiver
    })

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@internal
def _mint_to(
    to: address,
    token_id: uint256,
    metadata: NFTMetadata,
    attrs: DynArray[Attribute, 20]
):
    assert to != empty(address), "Mint to zero"
    assert self.token_owner[token_id] == empty(address), "Already minted"
    assert self.collection.minted_count < self.collection.max_supply, "Max supply"

    # Store token
    self.tokens[token_id] = NFTToken({
        token_id: token_id,
        owner: to,
        metadata: metadata,
        minted_at: block.timestamp,
        last_transferred: block.timestamp,
        transfer_count: 0,
        is_locked: False
    })
    self.attributes[token_id] = attrs

    # Update ownership
    self.token_owner[token_id] = to
    self.owned_tokens_index[token_id] = convert(len(self.owned_tokens[to]), uint256)
    self.owned_tokens[to].append(token_id)
    self.collection.minted_count += 1

    log Transfer(empty(address), to, token_id)
    log Minted(to, token_id)

@internal
def _transfer(from_addr: address, to: address, token_id: uint256):
    assert self.token_owner[token_id] == from_addr, "Not owner"
    assert to != empty(address), "Transfer to zero"
    assert not self.tokens[token_id].is_locked, "Token is locked"

    # Remove from sender's list (swap and pop)
    idx: uint256 = self.owned_tokens_index[token_id]
    n: uint256 = convert(len(self.owned_tokens[from_addr]), uint256)
    if idx < n - 1:
        last_token: uint256 = self.owned_tokens[from_addr][n - 1]
        self.owned_tokens[from_addr][idx] = last_token
        self.owned_tokens_index[last_token] = idx
    self.owned_tokens[from_addr].pop()

    # Add to recipient's list
    self.owned_tokens_index[token_id] = convert(len(self.owned_tokens[to]), uint256)
    self.owned_tokens[to].append(token_id)

    # Update ownership
    self.token_owner[token_id] = to
    self.tokens[token_id].owner = to
    self.tokens[token_id].last_transferred = block.timestamp
    self.tokens[token_id].transfer_count += 1

    # Clear approval
    self.token_approvals[token_id] = empty(address)

    log Transfer(from_addr, to, token_id)

@view
@internal
def _is_approved_or_owner(spender: address, token_id: uint256) -> bool:
    token_owner_addr: address = self.token_owner[token_id]
    return (
        spender == token_owner_addr or
        self.token_approvals[token_id] == spender or
        self.operator_approvals[token_owner_addr][spender]
    )

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Mint Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def mint(
    to: address,
    token_id: uint256,
    metadata: NFTMetadata,
    attrs: DynArray[Attribute, 20]
):
    """Minter-only mint with full metadata"""
    assert msg.sender == self.minter, "Not minter"
    self._mint_to(to, token_id, metadata, attrs)

@payable
@external
def public_mint(
    token_id: uint256,
    metadata: NFTMetadata,
    attrs: DynArray[Attribute, 20]
):
    """Public mint with payment"""
    assert self.is_public_mint, "Public mint not active"
    assert msg.value >= self.mint_price, "Insufficient payment"
    self._mint_to(msg.sender, token_id, metadata, attrs)
    # Refund excess
    if msg.value > self.mint_price:
        send(msg.sender, msg.value - self.mint_price)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Transfer Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def transfer_from(from_addr: address, to: address, token_id: uint256):
    assert self._is_approved_or_owner(msg.sender, token_id), "Not approved"
    self._transfer(from_addr, to, token_id)

@external
def approve(to: address, token_id: uint256):
    token_owner_addr: address = self.token_owner[token_id]
    assert msg.sender == token_owner_addr, "Not owner"
    assert to != token_owner_addr, "Approve to owner"
    self.token_approvals[token_id] = to
    log Approval(token_owner_addr, to, token_id)

@external
def set_approval_for_all(operator: address, approved: bool):
    assert operator != msg.sender, "Self approve"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Metadata Update
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def update_metadata(token_id: uint256, new_metadata: NFTMetadata):
    """Owner can update their token's metadata"""
    assert self.token_owner[token_id] == msg.sender, "Not owner"
    self.tokens[token_id].metadata = new_metadata
    log MetadataUpdate(token_id)

@external
def add_attribute(token_id: uint256, attr: Attribute):
    """Add attribute to token"""
    assert self.token_owner[token_id] == msg.sender, "Not owner"
    assert convert(len(self.attributes[token_id]), uint256) < MAX_ATTRIBUTES, "Max attrs"
    self.attributes[token_id].append(attr)
    log MetadataUpdate(token_id)

@external
def lock_token(token_id: uint256):
    """Lock token (soul-bound)"""
    assert self.token_owner[token_id] == msg.sender, "Not owner"
    self.tokens[token_id].is_locked = True

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def owner_of(token_id: uint256) -> address:
    owner_addr: address = self.token_owner[token_id]
    assert owner_addr != empty(address), "Token not minted"
    return owner_addr

@view
@external
def balance_of(account: address) -> uint256:
    return convert(len(self.owned_tokens[account]), uint256)

@view
@external
def get_token(token_id: uint256) -> NFTToken:
    assert self.token_owner[token_id] != empty(address), "Not minted"
    return self.tokens[token_id]

@view
@external
def get_metadata(token_id: uint256) -> NFTMetadata:
    return self.tokens[token_id].metadata

@view
@external
def get_attributes(token_id: uint256) -> DynArray[Attribute, 20]:
    return self.attributes[token_id]

@view
@external
def get_owned_tokens(account: address) -> DynArray[uint256, 1000]:
    return self.owned_tokens[account]

@view
@external
def get_collection_info() -> CollectionInfo:
    return self.collection

@view
@external
def get_approved(token_id: uint256) -> address:
    return self.token_approvals[token_id]

@view
@external
def is_approved_for_all(owner_addr: address, operator: address) -> bool:
    return self.operator_approvals[owner_addr][operator]

@view
@external
def royalty_info(token_id: uint256, sale_price: uint256) -> (address, uint256):
    royalty: uint256 = (sale_price * self.collection.royalty_bps) / 10000
    return self.collection.royalty_receiver, royalty

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Admin Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def set_minter(new_minter: address):
    assert msg.sender == self.owner, "Not owner"
    self.minter = new_minter

@external
def set_public_mint(active: bool, price: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.is_public_mint = active
    self.mint_price = price

@external
def withdraw():
    assert msg.sender == self.owner, "Not owner"
    send(self.owner, self.balance)
```

### Test Code

```python
# tests/test_nft_collection.py
import pytest
from brownie import NFTCollection, accounts, reverts

@pytest.fixture
def nft(accounts):
    return NFTCollection.deploy(
        "MyNFT",
        "MNFT",
        10000,        # max supply
        250,          # 2.5% royalty
        accounts[9],  # royalty receiver
        {'from': accounts[0]}
    )

def make_metadata(name="Test NFT"):
    return (
        name, "A test NFT", "ipfs://image", "https://example.com",
        "FF0000", ""
    )

def make_attrs():
    return [("Background", "Blue"), ("Eyes", "Green")]

class TestMinting:
    def test_mint(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        assert nft.owner_of(1) == accounts[1]
        assert nft.balance_of(accounts[1]) == 1

    def test_mint_updates_collection(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        info = nft.get_collection_info()
        assert info[3] == 1  # minted_count

    def test_get_attributes(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        attrs = nft.get_attributes(1)
        assert attrs[0][0] == "Background"
        assert attrs[0][1] == "Blue"

class TestTransfer:
    def test_transfer(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        nft.transfer_from(accounts[1], accounts[2], 1, {'from': accounts[1]})
        assert nft.owner_of(1) == accounts[2]
        assert nft.balance_of(accounts[1]) == 0
        assert nft.balance_of(accounts[2]) == 1

    def test_approval(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        nft.approve(accounts[2], 1, {'from': accounts[1]})
        nft.transfer_from(accounts[1], accounts[3], 1, {'from': accounts[2]})
        assert nft.owner_of(1) == accounts[3]

    def test_locked_token_cannot_transfer(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        nft.lock_token(1, {'from': accounts[1]})
        with reverts("Token is locked"):
            nft.transfer_from(accounts[1], accounts[2], 1, {'from': accounts[1]})

class TestRoyalty:
    def test_royalty_calculation(self, nft, accounts):
        nft.mint(accounts[1], 1, make_metadata(), make_attrs(), {'from': accounts[0]})
        receiver, amount = nft.royalty_info(1, 10000)
        assert receiver == accounts[9]
        assert amount == 250  # 2.5%
```

---

## สรุป

- ✅ **Struct declaration**: รวม types หลายชนิดเข้าด้วยกัน
- ✅ **Named initialization**: `Struct({field: value})`
- ✅ **Field access**: `struct_var.field_name`
- ✅ **Nested structs**: สร้าง hierarchical data
- ✅ **Memory copy**: การ assign struct สร้าง copy (ไม่ใช่ reference)
- ✅ **DynArray of structs**: จัดเก็บ collection ของ structs

## แบบฝึกหัด

1. **สร้าง** Employee Management System ด้วย nested structs
2. **เพิ่ม** Batch mint ให้ NFTCollection รับ array of structs
3. **สร้าง** DAO Proposal System ด้วย struct state machine
4. **ทดสอบ** Gas ของการเขียน struct field เดี่ยว vs ทั้ง struct
5. **สร้าง** Multi-token Portfolio ที่ track หลาย NFT collections

---

**ก่อนหน้า: [Part 010 - HashMap (Mapping)](part_010_hashmap.md)**  
**ต่อไป: [Part 012 - Events และ Logging](part_012_events.md)**
