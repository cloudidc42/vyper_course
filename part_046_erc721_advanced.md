# Part 046: Advanced ERC-721 (NFT)

## สารบัญ
1. [Overview](#overview)
2. [ERC-721 Standard](#standard)
3. [ERC-2981 Royalties](#royalties)
4. [Enumerable Extension](#enumerable)
5. [Metadata และ IPFS](#metadata)
6. [Batch Minting](#batch)
7. [AdvancedNFT Contract](#advanced-nft)
8. [Testing](#testing)

---

## 1. Overview {#overview}

ERC-721 เป็น standard สำหรับ Non-Fungible Tokens (NFTs) บน Ethereum แต่ละ token มี unique ID และ owner

### ความแตกต่างจาก ERC-20

| Feature | ERC-20 | ERC-721 |
|---------|--------|---------|
| Fungibility | Fungible (แทนกันได้) | Non-Fungible (unique) |
| Unit | Amount | Token ID |
| Transfer | `transfer(to, amount)` | `transferFrom(from, to, tokenId)` |
| Balance | จำนวน tokens | จำนวน NFTs ที่ถือ |

### Use Cases

- **Digital Art**: ชิ้นงานศิลปะดิจิทัล
- **Gaming**: ไอเทมในเกม
- **Real Estate**: ที่ดิน/ทรัพย์สินดิจิทัล
- **Membership**: การเป็นสมาชิก exclusive

---

## 2. ERC-721 Standard {#standard}

```vyper
# @version 0.4.0
# @title BasicNFT
# @notice ERC-721 implementation พื้นฐาน

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        frm: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

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

# ERC-165 interface IDs
INTERFACE_ID_ERC165: constant(bytes4) = 0x01ffc9a7
INTERFACE_ID_ERC721: constant(bytes4) = 0x80ac58cd
INTERFACE_ID_ERC721_METADATA: constant(bytes4) = 0x5b5e139f

# State
owner_of: HashMap[uint256, address]         # tokenId => owner
balances: HashMap[address, uint256]         # owner => balance
token_approvals: HashMap[uint256, address]  # tokenId => approved
operator_approvals: HashMap[address, HashMap[address, bool]]  # owner => operator => approved

@deploy
def __init__():
    pass

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return interfaceId in [
        INTERFACE_ID_ERC165,
        INTERFACE_ID_ERC721,
        INTERFACE_ID_ERC721_METADATA
    ]

@external
@view
def balanceOf(owner: address) -> uint256:
    assert owner != empty(address), "Zero address"
    return self.balances[owner]

@external
@view
def ownerOf(tokenId: uint256) -> address:
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    return owner

@external
@view
def getApproved(tokenId: uint256) -> address:
    assert self.owner_of[tokenId] != empty(address), "Nonexistent token"
    return self.token_approvals[tokenId]

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@internal
def _is_approved_or_owner(spender: address, tokenId: uint256) -> bool:
    owner: address = self.owner_of[tokenId]
    return (
        spender == owner or
        self.operator_approvals[owner][spender] or
        self.token_approvals[tokenId] == spender
    )

@internal
def _transfer(sender: address, to: address, tokenId: uint256):
    assert self.owner_of[tokenId] == sender, "Not owner"
    assert to != empty(address), "Zero address"
    
    # Clear approval
    if self.token_approvals[tokenId] != empty(address):
        self.token_approvals[tokenId] = empty(address)
    
    self.balances[sender] -= 1
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    
    log Transfer(sender, to, tokenId)

@external
def transferFrom(sender: address, to: address, tokenId: uint256):
    assert self._is_approved_or_owner(msg.sender, tokenId), "Not approved"
    self._transfer(sender, to, tokenId)

@external
def safeTransferFrom(
    sender: address,
    to: address,
    tokenId: uint256,
    data: Bytes[1024] = b""
):
    assert self._is_approved_or_owner(msg.sender, tokenId), "Not approved"
    self._transfer(sender, to, tokenId)
    
    if to.is_contract:
        result: bytes4 = IERC721Receiver(to).onERC721Received(
            msg.sender, sender, tokenId, data
        )
        assert result == method_id("onERC721Received(address,address,uint256,bytes)", output_type=bytes4), \
            "Invalid receiver"

@external
def approve(approved: address, tokenId: uint256):
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    assert msg.sender == owner or self.operator_approvals[owner][msg.sender], "Not authorized"
    
    self.token_approvals[tokenId] = approved
    log Approval(owner, approved, tokenId)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Approve to caller"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)
```

---

## 3. ERC-2981 Royalties {#royalties}

```vyper
# @version 0.4.0
# @title NFTWithRoyalties
# @notice ERC-721 + ERC-2981 (Royalty Standard)

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        frm: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

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

event RoyaltySet:
    token_id: indexed(uint256)
    recipient: address
    fee_numerator: uint256

# ERC-165 interface IDs
INTERFACE_ID_ERC165: constant(bytes4) = 0x01ffc9a7
INTERFACE_ID_ERC721: constant(bytes4) = 0x80ac58cd
INTERFACE_ID_ERC721_METADATA: constant(bytes4) = 0x5b5e139f
INTERFACE_ID_ERC2981: constant(bytes4) = 0x2a55205a

# Royalty denominator (10000 = 100%)
ROYALTY_DENOMINATOR: constant(uint256) = 10000

struct RoyaltyInfo:
    receiver: address
    royalty_fraction: uint256  # in basis points (1 = 0.01%)

# State
owner_of: HashMap[uint256, address]
balances: HashMap[address, uint256]
token_approvals: HashMap[uint256, address]
operator_approvals: HashMap[address, HashMap[address, bool]]

owner: public(address)
total_supply: public(uint256)
name: public(String[64])
symbol: public(String[32])

# Royalty info per token (can override default)
default_royalty: RoyaltyInfo
token_royalties: HashMap[uint256, RoyaltyInfo]
has_custom_royalty: HashMap[uint256, bool]

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    royalty_receiver: address,
    royalty_fraction: uint256
):
    self.owner = msg.sender
    self.name = name
    self.symbol = symbol
    
    assert royalty_fraction <= ROYALTY_DENOMINATOR, "Too high"
    self.default_royalty = RoyaltyInfo({
        receiver: royalty_receiver,
        royalty_fraction: royalty_fraction
    })

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return interfaceId in [
        INTERFACE_ID_ERC165,
        INTERFACE_ID_ERC721,
        INTERFACE_ID_ERC721_METADATA,
        INTERFACE_ID_ERC2981
    ]

# ========== ERC-2981 ==========

@external
@view
def royaltyInfo(
    tokenId: uint256,
    salePrice: uint256
) -> (address, uint256):
    """
    Returns royalty recipient and amount for a sale
    """
    royalty: RoyaltyInfo = self.default_royalty
    
    if self.has_custom_royalty[tokenId]:
        royalty = self.token_royalties[tokenId]
    
    if royalty.receiver == empty(address):
        return empty(address), 0
    
    royalty_amount: uint256 = salePrice * royalty.royalty_fraction // ROYALTY_DENOMINATOR
    return royalty.receiver, royalty_amount

@external
def setTokenRoyalty(
    tokenId: uint256,
    receiver: address,
    fraction: uint256
):
    assert msg.sender == self.owner, "Not owner"
    assert fraction <= ROYALTY_DENOMINATOR, "Too high"
    
    self.token_royalties[tokenId] = RoyaltyInfo({
        receiver: receiver,
        royalty_fraction: fraction
    })
    self.has_custom_royalty[tokenId] = True
    
    log RoyaltySet(tokenId, receiver, fraction)

@external
def setDefaultRoyalty(receiver: address, fraction: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert fraction <= ROYALTY_DENOMINATOR, "Too high"
    
    self.default_royalty = RoyaltyInfo({
        receiver: receiver,
        royalty_fraction: fraction
    })

# ========== ERC-721 ==========

@external
@view
def balanceOf(account: address) -> uint256:
    assert account != empty(address), "Zero address"
    return self.balances[account]

@external
@view
def ownerOf(tokenId: uint256) -> address:
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    return owner

@external
@view
def getApproved(tokenId: uint256) -> address:
    assert self.owner_of[tokenId] != empty(address), "Nonexistent token"
    return self.token_approvals[tokenId]

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@internal
def _is_approved_or_owner(spender: address, tokenId: uint256) -> bool:
    owner: address = self.owner_of[tokenId]
    return (
        spender == owner or
        self.operator_approvals[owner][spender] or
        self.token_approvals[tokenId] == spender
    )

@internal
def _mint(to: address, tokenId: uint256):
    assert to != empty(address), "Zero address"
    assert self.owner_of[tokenId] == empty(address), "Already minted"
    
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    self.total_supply += 1
    
    log Transfer(empty(address), to, tokenId)

@internal
def _transfer(sender: address, to: address, tokenId: uint256):
    assert self.owner_of[tokenId] == sender, "Not owner"
    assert to != empty(address), "Zero address"
    
    if self.token_approvals[tokenId] != empty(address):
        self.token_approvals[tokenId] = empty(address)
    
    self.balances[sender] -= 1
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    
    log Transfer(sender, to, tokenId)

@external
def mint(to: address):
    assert msg.sender == self.owner, "Not owner"
    token_id: uint256 = self.total_supply
    self._mint(to, token_id)

@external
def transferFrom(sender: address, to: address, tokenId: uint256):
    assert self._is_approved_or_owner(msg.sender, tokenId), "Not approved"
    self._transfer(sender, to, tokenId)

@external
def approve(approved: address, tokenId: uint256):
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    assert msg.sender == owner or self.operator_approvals[owner][msg.sender], "Not authorized"
    
    self.token_approvals[tokenId] = approved
    log Approval(owner, approved, tokenId)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Approve to caller"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)
```

---

## 4. AdvancedNFT Contract {#advanced-nft}

```vyper
# @version 0.4.0
# @title AdvancedNFT
# @notice Full-featured NFT with metadata, royalties, whitelist, and batch minting

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        frm: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

# ===== Events =====
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

event Minted:
    to: indexed(address)
    tokenId: uint256
    quantity: uint256

event BaseURIChanged:
    new_uri: String[256]

event SaleStateChanged:
    public_sale: bool
    whitelist_sale: bool

# ===== Constants =====
INTERFACE_ID_ERC165: constant(bytes4) = 0x01ffc9a7
INTERFACE_ID_ERC721: constant(bytes4) = 0x80ac58cd
INTERFACE_ID_ERC721_METADATA: constant(bytes4) = 0x5b5e139f
INTERFACE_ID_ERC2981: constant(bytes4) = 0x2a55205a

ROYALTY_DENOMINATOR: constant(uint256) = 10000
MAX_SUPPLY: constant(uint256) = 10000
MAX_BATCH_MINT: constant(uint256) = 20
MAX_PER_WALLET: constant(uint256) = 5

# ===== State =====
name: public(String[64])
symbol: public(String[32])
owner: public(address)

# ERC-721 core
owner_of: HashMap[uint256, address]
balances: HashMap[address, uint256]
token_approvals: HashMap[uint256, address]
operator_approvals: HashMap[address, HashMap[address, bool]]

# Supply tracking
total_supply: public(uint256)
next_token_id: uint256

# Metadata
base_uri: String[256]
base_uri_frozen: bool

# Royalties
royalty_receiver: public(address)
royalty_fraction: public(uint256)

# Sale config
public_sale_active: public(bool)
whitelist_sale_active: public(bool)
mint_price: public(uint256)  # in wei
whitelist_price: public(uint256)

# Whitelist
whitelist: public(HashMap[address, bool])
whitelist_minted: public(HashMap[address, uint256])
public_minted: public(HashMap[address, uint256])

# Treasury
treasury: public(address)

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    base_uri: String[256],
    mint_price: uint256,
    royalty_receiver: address,
    royalty_fraction: uint256,
    treasury: address
):
    self.owner = msg.sender
    self.name = name
    self.symbol = symbol
    self.base_uri = base_uri
    self.mint_price = mint_price
    self.whitelist_price = mint_price * 80 // 100  # 20% discount
    self.royalty_receiver = royalty_receiver
    assert royalty_fraction <= 1000, "Royalty too high (max 10%)"
    self.royalty_fraction = royalty_fraction
    self.treasury = treasury
    self.next_token_id = 1  # Start from 1

# ===== ERC-165 =====

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return interfaceId in [
        INTERFACE_ID_ERC165,
        INTERFACE_ID_ERC721,
        INTERFACE_ID_ERC721_METADATA,
        INTERFACE_ID_ERC2981
    ]

# ===== ERC-721 View =====

@external
@view
def balanceOf(account: address) -> uint256:
    assert account != empty(address), "Zero address"
    return self.balances[account]

@external
@view
def ownerOf(tokenId: uint256) -> address:
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    return owner

@external
@view
def getApproved(tokenId: uint256) -> address:
    assert self.owner_of[tokenId] != empty(address), "Nonexistent token"
    return self.token_approvals[tokenId]

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@external
@view
def tokenURI(tokenId: uint256) -> String[512]:
    assert self.owner_of[tokenId] != empty(address), "Nonexistent token"
    # ในการใช้งานจริง จะ concat base_uri + tokenId
    return self.base_uri

# ===== ERC-2981 =====

@external
@view
def royaltyInfo(tokenId: uint256, salePrice: uint256) -> (address, uint256):
    royalty_amount: uint256 = salePrice * self.royalty_fraction // ROYALTY_DENOMINATOR
    return self.royalty_receiver, royalty_amount

# ===== Internal Helpers =====

@internal
def _is_approved_or_owner(spender: address, tokenId: uint256) -> bool:
    owner: address = self.owner_of[tokenId]
    return (
        spender == owner or
        self.operator_approvals[owner][spender] or
        self.token_approvals[tokenId] == spender
    )

@internal
def _mint_single(to: address) -> uint256:
    """Mint single token and return tokenId"""
    assert self.total_supply < MAX_SUPPLY, "Sold out"
    
    token_id: uint256 = self.next_token_id
    self.next_token_id += 1
    self.total_supply += 1
    
    self.balances[to] += 1
    self.owner_of[token_id] = to
    
    log Transfer(empty(address), to, token_id)
    return token_id

@internal
def _check_wallet_limit(account: address, quantity: uint256, is_whitelist: bool):
    """Check minting limits per wallet"""
    if is_whitelist:
        assert self.whitelist_minted[account] + quantity <= MAX_PER_WALLET, \
            "Exceeds whitelist limit"
    else:
        assert self.public_minted[account] + quantity <= MAX_PER_WALLET, \
            "Exceeds public mint limit"

# ===== Minting Functions =====

@external
@payable
def publicMint(quantity: uint256):
    """Public sale minting"""
    assert self.public_sale_active, "Public sale not active"
    assert quantity > 0, "Quantity must be positive"
    assert quantity <= MAX_BATCH_MINT, "Exceeds batch limit"
    assert self.total_supply + quantity <= MAX_SUPPLY, "Would exceed max supply"
    
    self._check_wallet_limit(msg.sender, quantity, False)
    
    total_price: uint256 = self.mint_price * quantity
    assert msg.value >= total_price, "Insufficient ETH"
    
    for i: uint256 in range(20):
        if i >= quantity:
            break
        self._mint_single(msg.sender)
    
    self.public_minted[msg.sender] += quantity
    
    # Refund excess ETH
    if msg.value > total_price:
        send(msg.sender, msg.value - total_price)
    
    log Minted(msg.sender, self.next_token_id - quantity, quantity)

@external
@payable
def whitelistMint(quantity: uint256):
    """Whitelist sale minting (discounted)"""
    assert self.whitelist_sale_active, "Whitelist sale not active"
    assert self.whitelist[msg.sender], "Not whitelisted"
    assert quantity > 0, "Quantity must be positive"
    assert quantity <= MAX_BATCH_MINT, "Exceeds batch limit"
    assert self.total_supply + quantity <= MAX_SUPPLY, "Would exceed max supply"
    
    self._check_wallet_limit(msg.sender, quantity, True)
    
    total_price: uint256 = self.whitelist_price * quantity
    assert msg.value >= total_price, "Insufficient ETH"
    
    for i: uint256 in range(20):
        if i >= quantity:
            break
        self._mint_single(msg.sender)
    
    self.whitelist_minted[msg.sender] += quantity
    
    # Refund excess ETH
    if msg.value > total_price:
        send(msg.sender, msg.value - total_price)
    
    log Minted(msg.sender, self.next_token_id - quantity, quantity)

@external
def ownerMint(to: address, quantity: uint256):
    """Owner can mint for free (airdrops, team allocation)"""
    assert msg.sender == self.owner, "Not owner"
    assert quantity > 0, "Must mint at least 1"
    assert self.total_supply + quantity <= MAX_SUPPLY, "Would exceed max supply"
    
    for i: uint256 in range(20):
        if i >= quantity:
            break
        self._mint_single(to)
    
    log Minted(to, self.next_token_id - quantity, quantity)

# ===== ERC-721 Transfer Functions =====

@external
def transferFrom(sender: address, to: address, tokenId: uint256):
    assert self._is_approved_or_owner(msg.sender, tokenId), "Not approved"
    assert self.owner_of[tokenId] == sender, "Not owner"
    assert to != empty(address), "Zero address"
    
    if self.token_approvals[tokenId] != empty(address):
        self.token_approvals[tokenId] = empty(address)
    
    self.balances[sender] -= 1
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    
    log Transfer(sender, to, tokenId)

@external
def safeTransferFrom(
    sender: address,
    to: address,
    tokenId: uint256,
    data: Bytes[1024] = b""
):
    assert self._is_approved_or_owner(msg.sender, tokenId), "Not approved"
    assert self.owner_of[tokenId] == sender, "Not owner"
    assert to != empty(address), "Zero address"
    
    if self.token_approvals[tokenId] != empty(address):
        self.token_approvals[tokenId] = empty(address)
    
    self.balances[sender] -= 1
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    
    log Transfer(sender, to, tokenId)
    
    if to.is_contract:
        result: bytes4 = IERC721Receiver(to).onERC721Received(
            msg.sender, sender, tokenId, data
        )
        assert result == 0x150b7a02, "Invalid receiver"

@external
def approve(approved: address, tokenId: uint256):
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent token"
    assert msg.sender == owner or self.operator_approvals[owner][msg.sender], "Not authorized"
    
    self.token_approvals[tokenId] = approved
    log Approval(owner, approved, tokenId)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Approve to caller"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

# ===== Admin Functions =====

@external
def setSaleState(public_sale: bool, whitelist_sale: bool):
    assert msg.sender == self.owner, "Not owner"
    self.public_sale_active = public_sale
    self.whitelist_sale_active = whitelist_sale
    log SaleStateChanged(public_sale, whitelist_sale)

@external
def setMintPrice(price: uint256, whitelist_price: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.mint_price = price
    self.whitelist_price = whitelist_price

@external
def setBaseURI(uri: String[256]):
    assert msg.sender == self.owner, "Not owner"
    assert not self.base_uri_frozen, "URI frozen"
    self.base_uri = uri
    log BaseURIChanged(uri)

@external
def freezeBaseURI():
    assert msg.sender == self.owner, "Not owner"
    self.base_uri_frozen = True

@external
def addToWhitelist(accounts: DynArray[address, 500]):
    assert msg.sender == self.owner, "Not owner"
    for account: address in accounts:
        self.whitelist[account] = True

@external
def removeFromWhitelist(accounts: DynArray[address, 500]):
    assert msg.sender == self.owner, "Not owner"
    for account: address in accounts:
        self.whitelist[account] = False

@external
def setRoyalty(receiver: address, fraction: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert fraction <= 1000, "Royalty too high"
    self.royalty_receiver = receiver
    self.royalty_fraction = fraction

@external
def withdraw():
    assert msg.sender == self.owner or msg.sender == self.treasury, "Not authorized"
    amount: uint256 = self.balance
    assert amount > 0, "No balance"
    send(self.treasury, amount)

@external
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.owner = new_owner
```

---

## 5. Testing {#testing}

```python
# tests/test_advanced_nft.py
import pytest
import boa
from eth_utils import to_checksum_address

@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def treasury():
    return boa.env.generate_address()

@pytest.fixture
def nft(deployer, treasury):
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/AdvancedNFT.vy",
            "Advanced NFT",
            "ANFT",
            "ipfs://QmHash/",
            int(0.05 * 10**18),  # 0.05 ETH mint price
            deployer,             # royalty receiver
            500,                  # 5% royalty
            treasury             # treasury
        )

@pytest.fixture
def alice():
    addr = boa.env.generate_address()
    # Fund with ETH for minting
    boa.env.set_balance(addr, 10 * 10**18)
    return addr

@pytest.fixture
def bob():
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, 10 * 10**18)
    return addr

class TestAdvancedNFT:
    def test_deployment(self, nft, deployer):
        assert nft.name() == "Advanced NFT"
        assert nft.symbol() == "ANFT"
        assert nft.owner() == deployer
        assert nft.total_supply() == 0
    
    def test_owner_mint(self, nft, deployer, alice):
        with boa.env.prank(deployer):
            nft.ownerMint(alice, 3)
        
        assert nft.total_supply() == 3
        assert nft.balanceOf(alice) == 3
        
        # Token IDs start from 1
        assert nft.ownerOf(1) == alice
        assert nft.ownerOf(2) == alice
        assert nft.ownerOf(3) == alice
    
    def test_public_mint(self, nft, deployer, alice):
        mint_price = nft.mint_price()
        
        with boa.env.prank(deployer):
            nft.setSaleState(True, False)
        
        with boa.env.prank(alice):
            nft.publicMint(2, value=mint_price * 2)
        
        assert nft.total_supply() == 2
        assert nft.balanceOf(alice) == 2
    
    def test_whitelist_mint(self, nft, deployer, alice):
        whitelist_price = nft.whitelist_price()
        
        with boa.env.prank(deployer):
            nft.setSaleState(False, True)
            nft.addToWhitelist([alice])
        
        with boa.env.prank(alice):
            nft.whitelistMint(1, value=whitelist_price)
        
        assert nft.total_supply() == 1
        assert nft.whitelist_minted(alice) == 1
    
    def test_whitelist_discount(self, nft):
        mint_price = nft.mint_price()
        whitelist_price = nft.whitelist_price()
        assert whitelist_price < mint_price
        assert whitelist_price == mint_price * 80 // 100
    
    def test_public_sale_not_active(self, nft, alice):
        with pytest.raises(Exception):
            with boa.env.prank(alice):
                nft.publicMint(1, value=int(0.05 * 10**18))
    
    def test_not_whitelisted(self, nft, deployer, alice):
        with boa.env.prank(deployer):
            nft.setSaleState(False, True)
        
        with pytest.raises(Exception):
            with boa.env.prank(alice):
                nft.whitelistMint(1, value=nft.whitelist_price())
    
    def test_max_per_wallet(self, nft, deployer, alice):
        mint_price = nft.mint_price()
        
        with boa.env.prank(deployer):
            nft.setSaleState(True, False)
        
        with boa.env.prank(alice):
            nft.publicMint(5, value=mint_price * 5)
        
        with pytest.raises(Exception):
            with boa.env.prank(alice):
                nft.publicMint(1, value=mint_price)
    
    def test_royalty_info(self, nft, deployer):
        with boa.env.prank(deployer):
            nft.ownerMint(deployer, 1)
        
        sale_price = 1 * 10**18  # 1 ETH
        receiver, amount = nft.royaltyInfo(1, sale_price)
        
        assert receiver == deployer
        expected_royalty = sale_price * 500 // 10000  # 5%
        assert amount == expected_royalty
    
    def test_transfer_nft(self, nft, deployer, alice, bob):
        with boa.env.prank(deployer):
            nft.ownerMint(alice, 1)
        
        with boa.env.prank(alice):
            nft.transferFrom(alice, bob, 1)
        
        assert nft.ownerOf(1) == bob
        assert nft.balanceOf(alice) == 0
        assert nft.balanceOf(bob) == 1
    
    def test_approve_and_transfer(self, nft, deployer, alice, bob):
        with boa.env.prank(deployer):
            nft.ownerMint(alice, 1)
        
        with boa.env.prank(alice):
            nft.approve(bob, 1)
        
        assert nft.getApproved(1) == bob
        
        with boa.env.prank(bob):
            nft.transferFrom(alice, bob, 1)
        
        assert nft.ownerOf(1) == bob
        # Approval cleared after transfer
        assert nft.getApproved(1) == "0x0000000000000000000000000000000000000000"
    
    def test_set_approval_for_all(self, nft, deployer, alice, bob):
        with boa.env.prank(deployer):
            nft.ownerMint(alice, 3)
        
        with boa.env.prank(alice):
            nft.setApprovalForAll(bob, True)
        
        assert nft.isApprovedForAll(alice, bob) == True
        
        # Bob can transfer any of Alice's tokens
        with boa.env.prank(bob):
            nft.transferFrom(alice, bob, 1)
            nft.transferFrom(alice, bob, 2)
        
        assert nft.balanceOf(alice) == 1
        assert nft.balanceOf(bob) == 2
    
    def test_withdraw(self, nft, deployer, treasury, alice):
        mint_price = nft.mint_price()
        
        with boa.env.prank(deployer):
            nft.setSaleState(True, False)
        
        with boa.env.prank(alice):
            nft.publicMint(5, value=mint_price * 5)
        
        treasury_balance_before = boa.env.get_balance(treasury)
        
        with boa.env.prank(deployer):
            nft.withdraw()
        
        treasury_balance_after = boa.env.get_balance(treasury)
        assert treasury_balance_after > treasury_balance_before
    
    def test_freeze_base_uri(self, nft, deployer):
        with boa.env.prank(deployer):
            nft.setBaseURI("ipfs://NewHash/")
            nft.freezeBaseURI()
        
        with pytest.raises(Exception):
            with boa.env.prank(deployer):
                nft.setBaseURI("ipfs://AnotherHash/")
    
    def test_supports_interface(self, nft):
        # ERC-165
        assert nft.supportsInterface(bytes.fromhex("01ffc9a7"))
        # ERC-721
        assert nft.supportsInterface(bytes.fromhex("80ac58cd"))
        # ERC-721 Metadata
        assert nft.supportsInterface(bytes.fromhex("5b5e139f"))
        # ERC-2981
        assert nft.supportsInterface(bytes.fromhex("2a55205a"))


if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

---

## สรุป

### Key Features ของ AdvancedNFT

1. **ERC-721 Standard**: ครบถ้วนตาม standard
2. **ERC-2981 Royalties**: รองรับ royalty ใน marketplaces
3. **ERC-165**: บอก interfaces ที่รองรับ
4. **Whitelist Minting**: mint ราคาพิเศษสำหรับ whitelist
5. **Batch Minting**: mint หลายตัวพร้อมกัน
6. **Max Per Wallet**: จำกัดการ mint ต่อ wallet
7. **Frozen URI**: lock metadata ให้ immutable
8. **Treasury**: แยก treasury address จาก owner
