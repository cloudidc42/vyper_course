# Part 034: Auction Contract

## สารบัญ
1. [บทนำ Auction](#บทนำ)
2. [English Auction](#english-auction)
3. [Dutch Auction](#dutch-auction)
4. [Sealed Bid Auction](#sealed-bid-auction)
5. [NFT Auction](#nft-auction)
6. [ตัวอย่าง: Full Auction System](#ตัวอย่าง-full-auction-system)
7. [Test Code](#test-code)

---

## บทนำ

**Auction Contract** ช่วยสร้างระบบประมูลแบบ Decentralized ที่ไม่ต้องมีตัวกลาง

### ประเภท Auction
- **English Auction**: ราคาเพิ่มขึ้น ผู้ให้ราคาสูงสุดชนะ
- **Dutch Auction**: ราคาลดลงจากสูงไปต่ำ
- **Sealed Bid**: ประมูลแบบปิด
- **Vickrey**: Sealed bid แต่จ่ายราคาอันดับ 2

---

## English Auction

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title English Auction
# @notice ราคาเริ่มต้นต่ำ เพิ่มขึ้นเรื่อยๆ จนหมดเวลา

event AuctionStarted:
    auction_id: indexed(uint256)
    seller: indexed(address)
    start_price: uint256
    end_time: uint256

event BidPlaced:
    auction_id: indexed(uint256)
    bidder: indexed(address)
    amount: uint256

event AuctionEnded:
    auction_id: indexed(uint256)
    winner: indexed(address)
    final_price: uint256

event BidWithdrawn:
    bidder: indexed(address)
    amount: uint256

struct Auction:
    seller: address
    start_price: uint256
    min_increment: uint256    # ราคาขั้นต่ำที่ต้องเพิ่มขึ้น
    current_price: uint256
    highest_bidder: address
    end_time: uint256
    ended: bool
    item_description: String[200]

auction_count: public(uint256)
auctions: public(HashMap[uint256, Auction])
pending_returns: public(HashMap[address, uint256])  # refunds ที่รอดึง

EXTENSION_TIME: constant(uint256) = 300  # เพิ่มเวลา 5 นาทีถ้า bid ใกล้ end

@deploy
def __init__():
    pass

@external
def start_auction(
    start_price: uint256,
    min_increment: uint256,
    duration: uint256,
    description: String[200]
) -> uint256:
    """สร้าง auction ใหม่"""
    assert start_price > 0
    assert duration > 0 and duration <= 30 * 86400  # สูงสุด 30 วัน
    
    auction_id: uint256 = self.auction_count
    
    self.auctions[auction_id] = Auction({
        seller: msg.sender,
        start_price: start_price,
        min_increment: min_increment,
        current_price: start_price,
        highest_bidder: empty(address),
        end_time: block.timestamp + duration,
        ended: False,
        item_description: description
    })
    
    self.auction_count += 1
    
    log AuctionStarted(auction_id, msg.sender, start_price, block.timestamp + duration)
    return auction_id

@external
@payable
def place_bid(auction_id: uint256):
    """ประมูล"""
    auction: Auction = self.auctions[auction_id]
    
    assert not auction.ended, "Auction ended"
    assert block.timestamp < auction.end_time, "Auction expired"
    assert msg.sender != auction.seller, "Seller cannot bid"
    
    # ตรวจสอบราคาขั้นต่ำ
    min_bid: uint256 = auction.current_price + auction.min_increment
    if auction.highest_bidder == empty(address):
        min_bid = auction.start_price
    
    assert msg.value >= min_bid, "Bid too low"
    
    # Refund ผู้ประมูลก่อนหน้า
    if auction.highest_bidder != empty(address):
        self.pending_returns[auction.highest_bidder] += auction.current_price
    
    # อัปเดต auction
    self.auctions[auction_id].current_price = msg.value
    self.auctions[auction_id].highest_bidder = msg.sender
    
    # Anti-snipe: extend ถ้า bid ใกล้ end time
    if block.timestamp + EXTENSION_TIME >= auction.end_time:
        self.auctions[auction_id].end_time = block.timestamp + EXTENSION_TIME
    
    log BidPlaced(auction_id, msg.sender, msg.value)

@external
def end_auction(auction_id: uint256):
    """จบ auction"""
    auction: Auction = self.auctions[auction_id]
    
    assert block.timestamp >= auction.end_time, "Auction not ended"
    assert not auction.ended, "Already ended"
    
    self.auctions[auction_id].ended = True
    
    if auction.highest_bidder != empty(address):
        # ส่งเงินให้ seller
        send(auction.seller, auction.current_price)
        log AuctionEnded(auction_id, auction.highest_bidder, auction.current_price)
    else:
        # ไม่มีผู้ประมูล
        log AuctionEnded(auction_id, empty(address), 0)

@external
def withdraw():
    """ดึงเงิน refund"""
    amount: uint256 = self.pending_returns[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    
    self.pending_returns[msg.sender] = 0
    send(msg.sender, amount)
    
    log BidWithdrawn(msg.sender, amount)
```

---

## Dutch Auction

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Dutch Auction
# @notice ราคาเริ่มสูงแล้วลดลงตามเวลา ใครซื้อก่อนได้ก่อน

event DutchAuctionStarted:
    auction_id: indexed(uint256)
    seller: indexed(address)
    start_price: uint256
    end_price: uint256
    duration: uint256

event DutchAuctionSold:
    auction_id: indexed(uint256)
    buyer: indexed(address)
    price: uint256

struct DutchAuction:
    seller: address
    start_price: uint256    # ราคาเริ่มต้น (สูงสุด)
    end_price: uint256      # ราคาสุดท้าย (ต่ำสุด)
    start_time: uint256
    duration: uint256       # ระยะเวลาทั้งหมด
    quantity: uint256       # จำนวนที่ขาย
    sold: uint256           # จำนวนที่ขายแล้ว
    ended: bool

auction_count: public(uint256)
auctions: public(HashMap[uint256, DutchAuction])

@deploy
def __init__():
    pass

@internal
def _get_current_price(auction_id: uint256) -> uint256:
    """คำนวณราคาปัจจุบัน"""
    auction: DutchAuction = self.auctions[auction_id]
    
    if block.timestamp >= auction.start_time + auction.duration:
        return auction.end_price
    
    elapsed: uint256 = block.timestamp - auction.start_time
    price_drop: uint256 = (auction.start_price - auction.end_price) * elapsed / auction.duration
    
    return auction.start_price - price_drop

@external
def create_dutch_auction(
    start_price: uint256,
    end_price: uint256,
    duration: uint256,
    quantity: uint256
) -> uint256:
    """สร้าง Dutch Auction"""
    assert start_price > end_price, "Start must be > end"
    assert end_price > 0
    assert duration > 0
    assert quantity > 0
    
    auction_id: uint256 = self.auction_count
    
    self.auctions[auction_id] = DutchAuction({
        seller: msg.sender,
        start_price: start_price,
        end_price: end_price,
        start_time: block.timestamp,
        duration: duration,
        quantity: quantity,
        sold: 0,
        ended: False
    })
    
    self.auction_count += 1
    
    log DutchAuctionStarted(auction_id, msg.sender, start_price, end_price, duration)
    return auction_id

@external
@payable
def buy(auction_id: uint256, quantity: uint256):
    """ซื้อในราคาปัจจุบัน"""
    auction: DutchAuction = self.auctions[auction_id]
    
    assert not auction.ended
    assert auction.sold + quantity <= auction.quantity, "Not enough supply"
    
    current_price: uint256 = self._get_current_price(auction_id)
    total_cost: uint256 = current_price * quantity
    
    assert msg.value >= total_cost, "Insufficient payment"
    
    self.auctions[auction_id].sold += quantity
    
    if self.auctions[auction_id].sold >= auction.quantity:
        self.auctions[auction_id].ended = True
    
    # ส่งเงินให้ seller
    send(auction.seller, total_cost)
    
    # Refund ส่วนเกิน
    if msg.value > total_cost:
        send(msg.sender, msg.value - total_cost)
    
    log DutchAuctionSold(auction_id, msg.sender, current_price)

@view
@external
def get_current_price(auction_id: uint256) -> uint256:
    return self._get_current_price(auction_id)

@view
@external
def get_remaining(auction_id: uint256) -> uint256:
    auction: DutchAuction = self.auctions[auction_id]
    return auction.quantity - auction.sold
```

---

## Sealed Bid Auction

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Sealed Bid Auction
# @notice ประมูลแบบปิด - ไม่รู้ราคาของคนอื่น

event CommitmentSubmitted:
    auction_id: indexed(uint256)
    bidder: indexed(address)

event BidRevealed:
    auction_id: indexed(uint256)
    bidder: indexed(address)
    amount: uint256

event SealedAuctionEnded:
    auction_id: indexed(uint256)
    winner: indexed(address)
    amount: uint256

struct SealedAuction:
    seller: address
    commit_end: uint256    # หมดเวลายื่น
    reveal_end: uint256    # หมดเวลาเปิด
    highest_bid: uint256
    highest_bidder: address
    ended: bool

auction_count: public(uint256)
auctions: public(HashMap[uint256, SealedAuction])
commitments: HashMap[uint256, HashMap[address, bytes32]]  # hash ของ bid
revealed_bids: HashMap[uint256, HashMap[address, uint256]]
deposits: HashMap[uint256, HashMap[address, uint256]]  # deposit ที่จ่าย

@deploy
def __init__():
    pass

@external
def create_sealed_auction(
    commit_duration: uint256,
    reveal_duration: uint256
) -> uint256:
    """สร้าง sealed bid auction"""
    auction_id: uint256 = self.auction_count
    
    self.auctions[auction_id] = SealedAuction({
        seller: msg.sender,
        commit_end: block.timestamp + commit_duration,
        reveal_end: block.timestamp + commit_duration + reveal_duration,
        highest_bid: 0,
        highest_bidder: empty(address),
        ended: False
    })
    
    self.auction_count += 1
    return auction_id

@external
@payable
def commit_bid(auction_id: uint256, commitment: bytes32):
    """
    @notice ยื่น commitment (hash ของ bid)
    @param commitment = keccak256(abi.encodePacked(bid_amount, secret))
    @dev ต้องส่ง ETH มากกว่า bid จริง (ป้องกันการเดา)
    """
    auction: SealedAuction = self.auctions[auction_id]
    
    assert block.timestamp < auction.commit_end, "Commit period ended"
    assert msg.value > 0, "Must send deposit"
    
    self.commitments[auction_id][msg.sender] = commitment
    self.deposits[auction_id][msg.sender] += msg.value

@external
def reveal_bid(auction_id: uint256, bid_amount: uint256, secret: bytes32):
    """
    @notice เปิดเผย bid จริง
    @param bid_amount จำนวน bid จริง
    @param secret รหัสลับที่ใช้สร้าง commitment
    """
    auction: SealedAuction = self.auctions[auction_id]
    
    assert block.timestamp >= auction.commit_end, "Commit still active"
    assert block.timestamp < auction.reveal_end, "Reveal period ended"
    
    # ตรวจสอบ commitment
    commitment: bytes32 = keccak256(
        concat(convert(bid_amount, bytes32), secret)
    )
    assert commitment == self.commitments[auction_id][msg.sender], \
        "Invalid commitment"
    
    # ตรวจสอบ deposit พอ
    assert self.deposits[auction_id][msg.sender] >= bid_amount, \
        "Deposit insufficient for revealed bid"
    
    self.revealed_bids[auction_id][msg.sender] = bid_amount
    
    # อัปเดต highest bid
    if bid_amount > auction.highest_bid:
        # Refund ผู้ชนะก่อน (ถ้ามี)
        if auction.highest_bidder != empty(address):
            old_deposit: uint256 = self.deposits[auction_id][auction.highest_bidder]
            self.deposits[auction_id][auction.highest_bidder] = old_deposit - auction.highest_bid
            # Refund ส่วนที่เหลือ
        
        self.auctions[auction_id].highest_bid = bid_amount
        self.auctions[auction_id].highest_bidder = msg.sender
    
    log BidRevealed(auction_id, msg.sender, bid_amount)

@external
def end_sealed_auction(auction_id: uint256):
    """จบ auction หลัง reveal period"""
    auction: SealedAuction = self.auctions[auction_id]
    
    assert block.timestamp >= auction.reveal_end, "Reveal not ended"
    assert not auction.ended, "Already ended"
    
    self.auctions[auction_id].ended = True
    
    if auction.highest_bidder != empty(address):
        send(auction.seller, auction.highest_bid)
        log SealedAuctionEnded(auction_id, auction.highest_bidder, auction.highest_bid)
    
@external
def withdraw_deposit(auction_id: uint256):
    """ดึง deposit คืน (losers)"""
    auction: SealedAuction = self.auctions[auction_id]
    assert auction.ended, "Not ended"
    assert msg.sender != auction.highest_bidder, "Winner cannot withdraw here"
    
    deposit: uint256 = self.deposits[auction_id][msg.sender]
    self.deposits[auction_id][msg.sender] = 0
    send(msg.sender, deposit)
```

---

## NFT Auction

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title NFT Auction
# @notice ประมูล NFT แบบ English Auction

interface ERC721:
    def safeTransferFrom(from_: address, to: address, token_id: uint256): nonpayable
    def transferFrom(from_: address, to: address, token_id: uint256): nonpayable
    def ownerOf(token_id: uint256) -> address: view
    def approve(to: address, token_id: uint256): nonpayable

event NFTAuctionCreated:
    auction_id: indexed(uint256)
    nft_contract: indexed(address)
    token_id: uint256
    seller: indexed(address)
    start_price: uint256

event NFTBidPlaced:
    auction_id: indexed(uint256)
    bidder: indexed(address)
    amount: uint256

event NFTAuctionSettled:
    auction_id: indexed(uint256)
    winner: indexed(address)
    amount: uint256

struct NFTAuctionData:
    nft_contract: address
    token_id: uint256
    seller: address
    start_price: uint256
    reserve_price: uint256    # ราคาขั้นต่ำที่จะขาย
    current_bid: uint256
    highest_bidder: address
    end_time: uint256
    settled: bool

auction_count: public(uint256)
auctions: public(HashMap[uint256, NFTAuctionData])
pending_returns: HashMap[address, uint256]

PLATFORM_FEE_BPS: constant(uint256) = 250  # 2.5%
fee_collector: public(address)

@deploy
def __init__(_fee_collector: address):
    self.fee_collector = _fee_collector

@external
def create_nft_auction(
    nft_contract: address,
    token_id: uint256,
    start_price: uint256,
    reserve_price: uint256,
    duration: uint256
) -> uint256:
    """สร้าง NFT auction"""
    # ตรวจสอบว่าเป็นเจ้าของ NFT
    assert ERC721(nft_contract).ownerOf(token_id) == msg.sender, "Not NFT owner"
    assert reserve_price >= start_price
    
    # โอน NFT เข้า contract
    ERC721(nft_contract).transferFrom(msg.sender, self, token_id)
    
    auction_id: uint256 = self.auction_count
    
    self.auctions[auction_id] = NFTAuctionData({
        nft_contract: nft_contract,
        token_id: token_id,
        seller: msg.sender,
        start_price: start_price,
        reserve_price: reserve_price,
        current_bid: 0,
        highest_bidder: empty(address),
        end_time: block.timestamp + duration,
        settled: False
    })
    
    self.auction_count += 1
    
    log NFTAuctionCreated(auction_id, nft_contract, token_id, msg.sender, start_price)
    return auction_id

@external
@payable
def bid(auction_id: uint256):
    """ประมูล NFT"""
    auction: NFTAuctionData = self.auctions[auction_id]
    
    assert not auction.settled
    assert block.timestamp < auction.end_time, "Auction ended"
    assert msg.sender != auction.seller, "Seller cannot bid"
    
    min_bid: uint256 = auction.start_price
    if auction.highest_bidder != empty(address):
        min_bid = auction.current_bid * 105 / 100  # ต้องเพิ่ม 5%
    
    assert msg.value >= min_bid, "Bid too low"
    
    # Refund ผู้ประมูลก่อน
    if auction.highest_bidder != empty(address):
        self.pending_returns[auction.highest_bidder] += auction.current_bid
    
    self.auctions[auction_id].current_bid = msg.value
    self.auctions[auction_id].highest_bidder = msg.sender
    
    log NFTBidPlaced(auction_id, msg.sender, msg.value)

@external
def settle(auction_id: uint256):
    """จบ auction และส่ง NFT + เงิน"""
    auction: NFTAuctionData = self.auctions[auction_id]
    
    assert block.timestamp >= auction.end_time
    assert not auction.settled
    
    self.auctions[auction_id].settled = True
    
    if auction.highest_bidder != empty(address) and \
       auction.current_bid >= auction.reserve_price:
        # ขายสำเร็จ
        fee: uint256 = auction.current_bid * PLATFORM_FEE_BPS / 10000
        seller_amount: uint256 = auction.current_bid - fee
        
        # ส่งเงิน
        send(auction.seller, seller_amount)
        send(self.fee_collector, fee)
        
        # โอน NFT
        ERC721(auction.nft_contract).safeTransferFrom(
            self, auction.highest_bidder, auction.token_id
        )
        
        log NFTAuctionSettled(auction_id, auction.highest_bidder, auction.current_bid)
    else:
        # ไม่ถึง reserve หรือไม่มีผู้ประมูล
        if auction.highest_bidder != empty(address):
            self.pending_returns[auction.highest_bidder] += auction.current_bid
        
        # คืน NFT ให้ seller
        ERC721(auction.nft_contract).safeTransferFrom(
            self, auction.seller, auction.token_id
        )

@external
def withdraw():
    amount: uint256 = self.pending_returns[msg.sender]
    assert amount > 0
    self.pending_returns[msg.sender] = 0
    send(msg.sender, amount)
```

---

## ตัวอย่าง: Full Auction System

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Full Auction System
# @notice ระบบ Auction ครบถ้วนรองรับ ETH และ Token

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

# ==================== Constants ====================

PLATFORM_FEE_BPS: constant(uint256) = 200  # 2%
MAX_DURATION: constant(uint256) = 30 * 86400  # 30 วัน
MIN_DURATION: constant(uint256) = 3600  # 1 ชั่วโมง
BID_INCREMENT_BPS: constant(uint256) = 500  # 5% minimum increment

# ==================== Enums as Constants ====================

AUCTION_ETH: constant(uint8) = 0    # ประมูลด้วย ETH
AUCTION_TOKEN: constant(uint8) = 1  # ประมูลด้วย Token

STATUS_ACTIVE: constant(uint8) = 0
STATUS_ENDED: constant(uint8) = 1
STATUS_CANCELLED: constant(uint8) = 2

# ==================== Events ====================

event AuctionCreated:
    id: indexed(uint256)
    seller: indexed(address)
    auction_type: uint8
    start_price: uint256
    reserve_price: uint256
    end_time: uint256

event BidPlaced:
    id: indexed(uint256)
    bidder: indexed(address)
    amount: uint256
    new_end_time: uint256

event AuctionSettled:
    id: indexed(uint256)
    winner: indexed(address)
    amount: uint256
    seller_received: uint256
    fee: uint256

event AuctionCancelled:
    id: indexed(uint256)

event Withdrawal:
    account: indexed(address)
    amount: uint256

# ==================== Structs ====================

struct AuctionInfo:
    seller: address
    payment_token: address    # address(0) = ETH
    auction_type: uint8
    item_description: String[500]
    start_price: uint256
    reserve_price: uint256
    buy_now_price: uint256    # 0 = ไม่มี buy now
    current_bid: uint256
    highest_bidder: address
    start_time: uint256
    end_time: uint256
    extension_window: uint256  # ขยายเวลาถ้า bid ใกล้ end
    extension_time: uint256
    status: uint8
    bid_count: uint256

# ==================== State Variables ====================

owner: public(address)
fee_collector: public(address)

auction_count: public(uint256)
auctions: public(HashMap[uint256, AuctionInfo])

eth_pending: HashMap[address, uint256]
token_pending: HashMap[address, HashMap[address, uint256]]

# ==================== Constructor ====================

@deploy
def __init__(_fee_collector: address):
    self.owner = msg.sender
    self.fee_collector = _fee_collector

# ==================== Create Auction ====================

@external
def create_eth_auction(
    start_price: uint256,
    reserve_price: uint256,
    buy_now_price: uint256,
    duration: uint256,
    extension_window: uint256,
    extension_time: uint256,
    description: String[500]
) -> uint256:
    """
    @notice สร้าง auction ที่ประมูลด้วย ETH
    """
    assert start_price > 0
    assert reserve_price >= start_price
    assert duration >= MIN_DURATION and duration <= MAX_DURATION
    assert buy_now_price == 0 or buy_now_price > reserve_price
    
    auction_id: uint256 = self.auction_count
    end_time: uint256 = block.timestamp + duration
    
    self.auctions[auction_id] = AuctionInfo({
        seller: msg.sender,
        payment_token: empty(address),
        auction_type: AUCTION_ETH,
        item_description: description,
        start_price: start_price,
        reserve_price: reserve_price,
        buy_now_price: buy_now_price,
        current_bid: 0,
        highest_bidder: empty(address),
        start_time: block.timestamp,
        end_time: end_time,
        extension_window: extension_window,
        extension_time: extension_time,
        status: STATUS_ACTIVE,
        bid_count: 0
    })
    
    self.auction_count += 1
    
    log AuctionCreated(
        auction_id, msg.sender, AUCTION_ETH,
        start_price, reserve_price, end_time
    )
    return auction_id

@external
def create_token_auction(
    payment_token: address,
    start_price: uint256,
    reserve_price: uint256,
    buy_now_price: uint256,
    duration: uint256,
    description: String[500]
) -> uint256:
    """
    @notice สร้าง auction ที่ประมูลด้วย ERC20 Token
    """
    assert payment_token != empty(address)
    assert start_price > 0
    assert reserve_price >= start_price
    assert duration >= MIN_DURATION and duration <= MAX_DURATION
    
    auction_id: uint256 = self.auction_count
    end_time: uint256 = block.timestamp + duration
    
    self.auctions[auction_id] = AuctionInfo({
        seller: msg.sender,
        payment_token: payment_token,
        auction_type: AUCTION_TOKEN,
        item_description: description,
        start_price: start_price,
        reserve_price: reserve_price,
        buy_now_price: buy_now_price,
        current_bid: 0,
        highest_bidder: empty(address),
        start_time: block.timestamp,
        end_time: end_time,
        extension_window: 0,
        extension_time: 0,
        status: STATUS_ACTIVE,
        bid_count: 0
    })
    
    self.auction_count += 1
    
    log AuctionCreated(
        auction_id, msg.sender, AUCTION_TOKEN,
        start_price, reserve_price, end_time
    )
    return auction_id

# ==================== Bidding ====================

@external
@payable
@nonreentrant
def bid_eth(auction_id: uint256):
    """ประมูลด้วย ETH"""
    auction: AuctionInfo = self.auctions[auction_id]
    
    assert auction.status == STATUS_ACTIVE, "Not active"
    assert auction.auction_type == AUCTION_ETH, "Not ETH auction"
    assert block.timestamp < auction.end_time, "Ended"
    assert msg.sender != auction.seller, "Seller cannot bid"
    
    # ตรวจสอบ min bid
    min_bid: uint256 = self._get_min_bid(auction_id)
    assert msg.value >= min_bid, "Bid too low"
    
    # Refund ผู้ประมูลก่อน
    if auction.highest_bidder != empty(address):
        self.eth_pending[auction.highest_bidder] += auction.current_bid
    
    # อัปเดต
    self.auctions[auction_id].current_bid = msg.value
    self.auctions[auction_id].highest_bidder = msg.sender
    self.auctions[auction_id].bid_count += 1
    
    # Extend ถ้าใกล้ end
    new_end_time: uint256 = auction.end_time
    if auction.extension_window > 0:
        if block.timestamp + auction.extension_window >= auction.end_time:
            new_end_time = block.timestamp + auction.extension_time
            self.auctions[auction_id].end_time = new_end_time
    
    log BidPlaced(auction_id, msg.sender, msg.value, new_end_time)
    
    # Buy Now
    if auction.buy_now_price > 0 and msg.value >= auction.buy_now_price:
        self._settle_auction(auction_id)

@external
@nonreentrant
def bid_token(auction_id: uint256, amount: uint256):
    """ประมูลด้วย Token"""
    auction: AuctionInfo = self.auctions[auction_id]
    
    assert auction.status == STATUS_ACTIVE, "Not active"
    assert auction.auction_type == AUCTION_TOKEN, "Not token auction"
    assert block.timestamp < auction.end_time, "Ended"
    assert msg.sender != auction.seller, "Seller cannot bid"
    
    min_bid: uint256 = self._get_min_bid(auction_id)
    assert amount >= min_bid, "Bid too low"
    
    # โอน token เข้า contract
    ERC20(auction.payment_token).transferFrom(msg.sender, self, amount)
    
    # Refund ผู้ประมูลก่อน
    if auction.highest_bidder != empty(address):
        self.token_pending[auction.highest_bidder][auction.payment_token] += auction.current_bid
    
    self.auctions[auction_id].current_bid = amount
    self.auctions[auction_id].highest_bidder = msg.sender
    self.auctions[auction_id].bid_count += 1
    
    log BidPlaced(auction_id, msg.sender, amount, auction.end_time)

# ==================== Settlement ====================

@internal
def _get_min_bid(auction_id: uint256) -> uint256:
    auction: AuctionInfo = self.auctions[auction_id]
    
    if auction.highest_bidder == empty(address):
        return auction.start_price
    
    increment: uint256 = auction.current_bid * BID_INCREMENT_BPS / 10000
    return auction.current_bid + increment

@internal
def _settle_auction(auction_id: uint256):
    """Internal: settle auction"""
    auction: AuctionInfo = self.auctions[auction_id]
    
    self.auctions[auction_id].status = STATUS_ENDED
    
    if auction.highest_bidder == empty(address) or \
       auction.current_bid < auction.reserve_price:
        # ไม่ถึง reserve - คืนเงินผู้ประมูล
        if auction.highest_bidder != empty(address):
            if auction.auction_type == AUCTION_ETH:
                self.eth_pending[auction.highest_bidder] += auction.current_bid
            else:
                self.token_pending[auction.highest_bidder][auction.payment_token] += auction.current_bid
        return
    
    # คำนวณ fee
    fee: uint256 = auction.current_bid * PLATFORM_FEE_BPS / 10000
    seller_amount: uint256 = auction.current_bid - fee
    
    if auction.auction_type == AUCTION_ETH:
        send(auction.seller, seller_amount)
        send(self.fee_collector, fee)
    else:
        ERC20(auction.payment_token).transfer(auction.seller, seller_amount)
        ERC20(auction.payment_token).transfer(self.fee_collector, fee)
    
    log AuctionSettled(
        auction_id,
        auction.highest_bidder,
        auction.current_bid,
        seller_amount,
        fee
    )

@external
@nonreentrant
def settle(auction_id: uint256):
    """จบ auction (manual settle)"""
    auction: AuctionInfo = self.auctions[auction_id]
    
    assert block.timestamp >= auction.end_time, "Not ended"
    assert auction.status == STATUS_ACTIVE, "Not active"
    
    self._settle_auction(auction_id)

@external
def cancel_auction(auction_id: uint256):
    """ยกเลิก auction"""
    auction: AuctionInfo = self.auctions[auction_id]
    
    assert msg.sender == auction.seller or msg.sender == self.owner
    assert auction.status == STATUS_ACTIVE
    assert auction.bid_count == 0, "Cannot cancel with bids"
    
    self.auctions[auction_id].status = STATUS_CANCELLED
    
    log AuctionCancelled(auction_id)

# ==================== Withdraw ====================

@external
@nonreentrant
def withdraw_eth():
    amount: uint256 = self.eth_pending[msg.sender]
    assert amount > 0
    self.eth_pending[msg.sender] = 0
    send(msg.sender, amount)
    log Withdrawal(msg.sender, amount)

@external
@nonreentrant
def withdraw_token(token: address):
    amount: uint256 = self.token_pending[msg.sender][token]
    assert amount > 0
    self.token_pending[msg.sender][token] = 0
    ERC20(token).transfer(msg.sender, amount)
    log Withdrawal(msg.sender, amount)

# ==================== View Functions ====================

@view
@external
def get_auction(auction_id: uint256) -> AuctionInfo:
    return self.auctions[auction_id]

@view
@external
def get_min_bid(auction_id: uint256) -> uint256:
    return self._get_min_bid(auction_id)

@view
@external
def get_pending_eth(account: address) -> uint256:
    return self.eth_pending[account]

@view
@external
def get_pending_token(account: address, token: address) -> uint256:
    return self.token_pending[account][token]

@view
@external
def is_active(auction_id: uint256) -> bool:
    auction: AuctionInfo = self.auctions[auction_id]
    return (auction.status == STATUS_ACTIVE and 
            block.timestamp < auction.end_time)
```

---

## Test Code

```python
# tests/test_auction.py
import pytest
import ape

ONE_HOUR = 3600

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def seller(accounts):
    return accounts[1]

@pytest.fixture
def bidder1(accounts):
    return accounts[2]

@pytest.fixture
def bidder2(accounts):
    return accounts[3]

@pytest.fixture
def fee_collector(accounts):
    return accounts[4]

@pytest.fixture
def auction_system(owner, fee_collector, project):
    return project.FullAuctionSystem.deploy(
        fee_collector.address,
        sender=owner
    )

class TestEnglishAuction:
    
    def test_create_auction(self, auction_system, seller):
        """ทดสอบสร้าง auction"""
        start_price = 1 * 10**18
        reserve_price = 2 * 10**18
        
        auction_id = auction_system.create_eth_auction(
            start_price,
            reserve_price,
            0,           # no buy now
            ONE_HOUR,    # 1 hour
            0, 0,
            "Test item",
            sender=seller
        )
        
        auction = auction_system.get_auction(auction_id)
        assert auction.seller == seller.address
        assert auction.start_price == start_price
    
    def test_place_bid(self, auction_system, seller, bidder1):
        """ทดสอบประมูล"""
        auction_id = auction_system.create_eth_auction(
            10**18, 2 * 10**18, 0, ONE_HOUR, 0, 0, "Item",
            sender=seller
        )
        
        auction_system.bid_eth(auction_id, sender=bidder1, value=10**18)
        
        auction = auction_system.get_auction(auction_id)
        assert auction.highest_bidder == bidder1.address
        assert auction.current_bid == 10**18
    
    def test_outbid(self, auction_system, seller, bidder1, bidder2):
        """ทดสอบ outbid"""
        auction_id = auction_system.create_eth_auction(
            10**18, 2 * 10**18, 0, ONE_HOUR, 0, 0, "Item",
            sender=seller
        )
        
        auction_system.bid_eth(auction_id, sender=bidder1, value=10**18)
        
        # Bidder2 outbids
        bid2 = 10**18 * 105 // 100 + 1  # 5% more + 1
        auction_system.bid_eth(auction_id, sender=bidder2, value=bid2)
        
        auction = auction_system.get_auction(auction_id)
        assert auction.highest_bidder == bidder2.address
        
        # Bidder1 should have pending return
        assert auction_system.get_pending_eth(bidder1.address) == 10**18
    
    def test_settle_above_reserve(self, auction_system, seller, bidder1, fee_collector, chain):
        """ทดสอบ settle เมื่อถึง reserve"""
        reserve = 2 * 10**18
        bid_amount = 3 * 10**18
        
        auction_id = auction_system.create_eth_auction(
            10**18, reserve, 0, ONE_HOUR, 0, 0, "Item",
            sender=seller
        )
        
        auction_system.bid_eth(auction_id, sender=bidder1, value=bid_amount)
        
        chain.mine(deltatime=ONE_HOUR + 1)
        
        before_seller = seller.balance
        before_fee = fee_collector.balance
        
        auction_system.settle(auction_id, sender=bidder1)
        
        expected_fee = bid_amount * 200 // 10000  # 2%
        assert fee_collector.balance - before_fee == expected_fee
        assert seller.balance - before_seller == bid_amount - expected_fee
    
    def test_bid_below_minimum_fails(self, auction_system, seller, bidder1):
        """ทดสอบ bid ต่ำเกินไป"""
        auction_id = auction_system.create_eth_auction(
            10**18, 2 * 10**18, 0, ONE_HOUR, 0, 0, "Item",
            sender=seller
        )
        
        with pytest.raises(Exception):
            auction_system.bid_eth(auction_id, sender=bidder1, value=10**18 - 1)
```

---

## สรุป

ระบบ Auction ใน Vyper:

| Auction Type | ลักษณะ | Use Case |
|-------------|--------|---------|
| English | ราคาขึ้น | NFT, ของหายาก |
| Dutch | ราคาลง | Token sale, IPO |
| Sealed Bid | ปิด | ที่ดิน, B2B |
| NFT Auction | English + NFT | NFT marketplace |

---

[⬅️ Part 033: Voting System](part_033_voting.md) | [Part 035: Staking Contract ➡️](part_035_staking.md)
