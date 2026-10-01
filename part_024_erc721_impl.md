# Part 024: ERC-721 Implementation

## สารบัญ
1. [Full ERC-721 Implementation](#full-erc721)
2. [Metadata และ Enumerable Extension](#metadata-enumerable)
3. [Royalties (EIP-2981)](#royalties)
4. [Complete Test Suite](#test-suite)
5. [แบบฝึกหัด](#exercises)

---

## 1. Full ERC-721 Implementation {#full-erc721}

```vyper
# @version 0.4.0
"""
@title Full ERC-721 NFT Implementation
@notice สมบูรณ์ตาม EIP-721, EIP-721-Metadata, EIP-165
@dev รองรับ Mint, Burn, Safe Transfer, Royalties
"""

# ==================== Interfaces ====================

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        sender: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

# ==================== Events ====================

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
    tokenId: indexed(uint256)
    uri: String[512]

event Burned:
    owner: indexed(address)
    tokenId: indexed(uint256)

# ==================== Constants ====================

ERC721_RECEIVED: constant(bytes4) = 0x150b7a02
ERC165_INTERFACE_ID: constant(bytes4) = 0x01ffc9a7
ERC721_INTERFACE_ID: constant(bytes4) = 0x80ac58cd
ERC721_METADATA_INTERFACE_ID: constant(bytes4) = 0x5b5e139f

MAX_TOKEN_URI_LENGTH: constant(uint256) = 512

# ==================== State Variables ====================

name: public(String[100])
symbol: public(String[32])

owner: public(address)
paused: public(bool)
minters: public(HashMap[address, bool])

# Token Data
tokenOwners: HashMap[uint256, address]
tokenApprovals: HashMap[uint256, address]
ownerTokenCount: HashMap[address, uint256]
operatorApprovals: HashMap[address, HashMap[address, bool]]
tokenURIs: HashMap[uint256, String[512]]

# Supply Tracking
totalSupply: public(uint256)
nextTokenId: public(uint256)
burned: HashMap[uint256, bool]

# Base URI
baseURI: String[256]

@deploy
def __init__(
    _name: String[100],
    _symbol: String[32],
    _baseURI: String[256]
):
    """
    @param _name ชื่อ Collection
    @param _symbol สัญลักษณ์
    @param _baseURI Base URI สำหรับ Metadata
    """
    assert len(_name) > 0, "Empty name"
    assert len(_symbol) > 0, "Empty symbol"
    
    self.name = _name
    self.symbol = _symbol
    self.baseURI = _baseURI
    self.owner = msg.sender
    self.minters[msg.sender] = True
    self.nextTokenId = 1  # เริ่มจาก 1

# ==================== ERC-165 ====================

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return (
        interfaceId == ERC165_INTERFACE_ID or
        interfaceId == ERC721_INTERFACE_ID or
        interfaceId == ERC721_METADATA_INTERFACE_ID
    )

# ==================== ERC-721 Core ====================

@external
@view
def balanceOf(_owner: address) -> uint256:
    assert _owner != empty(address), "Zero address"
    return self.ownerTokenCount[_owner]

@external
@view
def ownerOf(tokenId: uint256) -> address:
    tokenOwner: address = self.tokenOwners[tokenId]
    assert tokenOwner != empty(address), "Token does not exist"
    return tokenOwner

@external
def approve(approved: address, tokenId: uint256):
    """
    @notice อนุมัติ approved ให้จัดการ tokenId
    """
    tokenOwner: address = self.tokenOwners[tokenId]
    assert tokenOwner != empty(address), "Token does not exist"
    assert approved != tokenOwner, "Approve to owner"
    assert (
        msg.sender == tokenOwner or
        self.operatorApprovals[tokenOwner][msg.sender]
    ), "Not owner or operator"
    
    self.tokenApprovals[tokenId] = approved
    log Approval(tokenOwner, approved, tokenId)

@external
@view
def getApproved(tokenId: uint256) -> address:
    assert self.tokenOwners[tokenId] != empty(address), "Token does not exist"
    return self.tokenApprovals[tokenId]

@external
def setApprovalForAll(operator: address, approved: bool):
    """
    @notice อนุมัติ/ยกเลิก Operator
    """
    assert operator != msg.sender, "Approve to self"
    assert operator != empty(address), "Zero address"
    
    self.operatorApprovals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

@external
@view
def isApprovedForAll(_owner: address, operator: address) -> bool:
    return self.operatorApprovals[_owner][operator]

@external
def transferFrom(sender: address, receiver: address, tokenId: uint256):
    """
    @notice Transfer NFT (ไม่ Check Receiver Contract)
    """
    assert not self.paused, "Contract paused"
    assert self._isApprovedOrOwner(msg.sender, tokenId), "Not authorized"
    self._transfer(sender, receiver, tokenId)

@external
def safeTransferFrom(
    sender: address,
    receiver: address,
    tokenId: uint256,
    data: Bytes[1024] = b""
):
    """
    @notice Transfer NFT อย่างปลอดภัย (Check Receiver Contract)
    """
    assert not self.paused, "Contract paused"
    assert self._isApprovedOrOwner(msg.sender, tokenId), "Not authorized"
    self._safeTransfer(sender, receiver, tokenId, data)

# ==================== Metadata ====================

@external
@view
def tokenURI(tokenId: uint256) -> String[512]:
    """
    @notice ดู URI ของ Token Metadata
    """
    assert self.tokenOwners[tokenId] != empty(address), "Token does not exist"
    
    # ถ้ามี Individual URI ให้ใช้ก่อน
    individualURI: String[512] = self.tokenURIs[tokenId]
    if len(individualURI) > 0:
        return individualURI
    
    # ไม่งั้นใช้ Base URI + tokenId
    if len(self.baseURI) > 0:
        return concat(self.baseURI, uint2str(tokenId))
    
    return ""

# ==================== Internal Functions ====================

@internal
@view
def _exists(tokenId: uint256) -> bool:
    return self.tokenOwners[tokenId] != empty(address)

@internal
@view
def _isApprovedOrOwner(spender: address, tokenId: uint256) -> bool:
    tokenOwner: address = self.tokenOwners[tokenId]
    assert tokenOwner != empty(address), "Token does not exist"
    
    return (
        spender == tokenOwner or
        self.tokenApprovals[tokenId] == spender or
        self.operatorApprovals[tokenOwner][spender]
    )

@internal
def _transfer(sender: address, receiver: address, tokenId: uint256):
    """Core Transfer Logic"""
    assert self.tokenOwners[tokenId] == sender, "Not token owner"
    assert receiver != empty(address), "Transfer to zero address"
    
    # Clear Approval
    if self.tokenApprovals[tokenId] != empty(address):
        self.tokenApprovals[tokenId] = empty(address)
        log Approval(sender, empty(address), tokenId)
    
    # Update Ownership
    self.ownerTokenCount[sender] -= 1
    self.ownerTokenCount[receiver] += 1
    self.tokenOwners[tokenId] = receiver
    
    log Transfer(sender, receiver, tokenId)

@internal
def _safeTransfer(
    sender: address,
    receiver: address,
    tokenId: uint256,
    data: Bytes[1024]
):
    """Safe Transfer ที่ Check Receiver Contract"""
    self._transfer(sender, receiver, tokenId)
    
    if receiver.code_size > 0:
        returnValue: bytes4 = IERC721Receiver(receiver).onERC721Received(
            msg.sender, sender, tokenId, data
        )
        assert returnValue == ERC721_RECEIVED, "Transfer to non-receiver"

@internal
def _mint(to: address, tokenId: uint256, uri: String[512]):
    """Internal Mint"""
    assert to != empty(address), "Mint to zero address"
    assert not self._exists(tokenId), "Token already minted"
    
    self.tokenOwners[tokenId] = to
    self.ownerTokenCount[to] += 1
    self.totalSupply += 1
    
    if len(uri) > 0:
        self.tokenURIs[tokenId] = uri
    
    log Transfer(empty(address), to, tokenId)
    log Minted(to, tokenId, uri)

@internal
def _burn(tokenId: uint256):
    """Internal Burn"""
    tokenOwner: address = self.tokenOwners[tokenId]
    assert tokenOwner != empty(address), "Token does not exist"
    
    # Clear Approvals
    if self.tokenApprovals[tokenId] != empty(address):
        self.tokenApprovals[tokenId] = empty(address)
    
    # Update State
    self.ownerTokenCount[tokenOwner] -= 1
    self.tokenOwners[tokenId] = empty(address)
    self.totalSupply -= 1
    self.burned[tokenId] = True
    
    # Clear URI
    if len(self.tokenURIs[tokenId]) > 0:
        self.tokenURIs[tokenId] = ""
    
    log Transfer(tokenOwner, empty(address), tokenId)
    log Burned(tokenOwner, tokenId)

# ==================== Admin Functions ====================

@external
def mint(to: address, uri: String[512]) -> uint256:
    """
    @notice Mint NFT ใหม่
    @return tokenId ของ NFT ที่ Mint
    """
    assert self.minters[msg.sender], "Not minter"
    assert to != empty(address), "Zero address"
    
    tokenId: uint256 = self.nextTokenId
    self.nextTokenId += 1
    
    self._mint(to, tokenId, uri)
    return tokenId

@external
def mintBatch(
    to: address,
    count: uint256,
    baseTokenURI: String[512]
) -> uint256:
    """
    @notice Mint NFT หลายชิ้นพร้อมกัน
    @return tokenId แรกที่ Mint
    """
    assert self.minters[msg.sender], "Not minter"
    assert to != empty(address), "Zero address"
    assert count > 0, "Count must be positive"
    assert count <= 100, "Too many tokens"
    
    firstTokenId: uint256 = self.nextTokenId
    
    for i: uint256 in range(100):
        if i >= count:
            break
        
        tokenId: uint256 = self.nextTokenId
        self.nextTokenId += 1
        
        uri: String[512] = concat(baseTokenURI, uint2str(tokenId))
        self._mint(to, tokenId, uri)
    
    return firstTokenId

@external
def burn(tokenId: uint256):
    """
    @notice Burn NFT (เฉพาะ Owner หรือ Approved)
    """
    assert self._isApprovedOrOwner(msg.sender, tokenId), "Not authorized"
    self._burn(tokenId)

@external
def setTokenURI(tokenId: uint256, uri: String[512]):
    """
    @notice Set URI สำหรับ Token
    """
    assert msg.sender == self.owner, "Not owner"
    assert self._exists(tokenId), "Token does not exist"
    self.tokenURIs[tokenId] = uri

@external
def setBaseURI(uri: String[256]):
    assert msg.sender == self.owner, "Not owner"
    self.baseURI = uri

@external
def pause():
    assert msg.sender == self.owner, "Not owner"
    assert not self.paused, "Already paused"
    self.paused = True

@external
def unpause():
    assert msg.sender == self.owner, "Not owner"
    assert self.paused, "Not paused"
    self.paused = False

@external
def addMinter(minter: address):
    assert msg.sender == self.owner, "Not owner"
    assert minter != empty(address), "Zero address"
    self.minters[minter] = True

@external
def removeMinter(minter: address):
    assert msg.sender == self.owner, "Not owner"
    self.minters[minter] = False

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self.owner, "Not owner"
    assert newOwner != empty(address), "Zero address"
    self.owner = newOwner
```

---

## 2. Metadata และ Enumerable Extension {#metadata-enumerable}

```vyper
# @version 0.4.0
"""
@title ERC-721 Enumerable Extension
@notice รองรับการ List Tokens ทั้งหมดและ Tokens ของ Owner
"""

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    tokenId: indexed(uint256)

# Core Storage
tokenOwners: HashMap[uint256, address]
ownerTokenCount: HashMap[address, uint256]

# Enumerable Storage
allTokens: DynArray[uint256, 10000]
allTokensIndex: HashMap[uint256, uint256]  # tokenId → index in allTokens

# Owner's tokens
ownedTokens: HashMap[address, DynArray[uint256, 1000]]
ownedTokensIndex: HashMap[uint256, uint256]  # tokenId → index in ownedTokens

totalSupply: public(uint256)
nextTokenId: uint256

@deploy
def __init__():
    self.nextTokenId = 1

# ==================== Enumerable ====================

@external
@view
def tokenByIndex(index: uint256) -> uint256:
    """
    @notice ดู tokenId ตาม Global Index
    """
    assert index < len(self.allTokens), "Index out of bounds"
    return self.allTokens[index]

@external
@view
def tokenOfOwnerByIndex(_owner: address, index: uint256) -> uint256:
    """
    @notice ดู tokenId ของ Owner ตาม Index
    """
    assert index < len(self.ownedTokens[_owner]), "Index out of bounds"
    return self.ownedTokens[_owner][index]

@external
@view
def tokensOfOwner(_owner: address) -> DynArray[uint256, 1000]:
    """
    @notice ดู Tokens ทั้งหมดของ Owner
    """
    return self.ownedTokens[_owner]

# ==================== Internal Helpers ====================

@internal
def _addTokenToOwnerEnumeration(to: address, tokenId: uint256):
    """เพิ่ม Token เข้า Owner's List"""
    self.ownedTokensIndex[tokenId] = len(self.ownedTokens[to])
    self.ownedTokens[to].append(tokenId)

@internal
def _addTokenToAllTokensEnumeration(tokenId: uint256):
    """เพิ่ม Token เข้า Global List"""
    self.allTokensIndex[tokenId] = len(self.allTokens)
    self.allTokens.append(tokenId)

@internal
def _removeTokenFromOwnerEnumeration(from_: address, tokenId: uint256):
    """ลบ Token ออกจาก Owner's List (Swap and Pop)"""
    lastTokenIndex: uint256 = len(self.ownedTokens[from_]) - 1
    tokenIndex: uint256 = self.ownedTokensIndex[tokenId]
    
    if tokenIndex != lastTokenIndex:
        lastTokenId: uint256 = self.ownedTokens[from_][lastTokenIndex]
        self.ownedTokens[from_][tokenIndex] = lastTokenId
        self.ownedTokensIndex[lastTokenId] = tokenIndex
    
    self.ownedTokensIndex[tokenId] = 0
    self.ownedTokens[from_].pop()

@internal
def _mint(to: address) -> uint256:
    tokenId: uint256 = self.nextTokenId
    self.nextTokenId += 1
    
    self.tokenOwners[tokenId] = to
    self.ownerTokenCount[to] += 1
    self.totalSupply += 1
    
    self._addTokenToAllTokensEnumeration(tokenId)
    self._addTokenToOwnerEnumeration(to, tokenId)
    
    log Transfer(empty(address), to, tokenId)
    return tokenId

@internal
def _transfer(from_: address, to: address, tokenId: uint256):
    assert self.tokenOwners[tokenId] == from_
    assert to != empty(address)
    
    self._removeTokenFromOwnerEnumeration(from_, tokenId)
    self._addTokenToOwnerEnumeration(to, tokenId)
    
    self.ownerTokenCount[from_] -= 1
    self.ownerTokenCount[to] += 1
    self.tokenOwners[tokenId] = to
    
    log Transfer(from_, to, tokenId)

@external
def mint(to: address) -> uint256:
    return self._mint(to)

@external
def transferFrom(sender: address, receiver: address, tokenId: uint256):
    assert self.tokenOwners[tokenId] == sender or \
           msg.sender == self.tokenOwners[tokenId], "Not authorized"
    self._transfer(sender, receiver, tokenId)

@external
@view
def ownerOf(tokenId: uint256) -> address:
    owner: address = self.tokenOwners[tokenId]
    assert owner != empty(address), "Token does not exist"
    return owner

@external
@view
def balanceOf(_owner: address) -> uint256:
    return self.ownerTokenCount[_owner]
```

---

## 3. Royalties (EIP-2981) {#royalties}

EIP-2981 เป็นมาตรฐานสำหรับ NFT Royalties

```vyper
# @version 0.4.0
"""
@title ERC-721 with EIP-2981 Royalties
@notice NFT ที่มี Royalties ให้ Creator
"""

# ==================== Events ====================

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    tokenId: indexed(uint256)

event RoyaltySet:
    tokenId: indexed(uint256)
    receiver: indexed(address)
    royaltyBps: uint256

# ==================== Structs ====================

struct RoyaltyInfo:
    receiver: address
    royaltyBps: uint256  # Basis Points (100 = 1%, 10000 = 100%)

# ==================== State ====================

tokenOwners: HashMap[uint256, address]
ownerTokenCount: HashMap[address, uint256]
totalSupply: public(uint256)
nextTokenId: uint256

name: public(String[100])
symbol: public(String[32])

owner: address

# Royalty State
defaultRoyalty: RoyaltyInfo
tokenRoyalties: HashMap[uint256, RoyaltyInfo]

MAX_ROYALTY_BPS: constant(uint256) = 1000  # สูงสุด 10%

ERC2981_INTERFACE_ID: constant(bytes4) = 0x2a55205a
ERC721_INTERFACE_ID: constant(bytes4) = 0x80ac58cd
ERC165_INTERFACE_ID: constant(bytes4) = 0x01ffc9a7

@deploy
def __init__(
    _name: String[100],
    _symbol: String[32],
    royaltyReceiver: address,
    royaltyBps: uint256
):
    assert royaltyBps <= MAX_ROYALTY_BPS, "Royalty too high"
    assert royaltyReceiver != empty(address), "Zero receiver"
    
    self.name = _name
    self.symbol = _symbol
    self.owner = msg.sender
    self.nextTokenId = 1
    
    self.defaultRoyalty = RoyaltyInfo({
        receiver: royaltyReceiver,
        royaltyBps: royaltyBps
    })

# ==================== EIP-2981 ====================

@external
@view
def royaltyInfo(
    tokenId: uint256,
    salePrice: uint256
) -> (address, uint256):
    """
    @notice คำนวณ Royalty สำหรับ Sale
    @param tokenId ID ของ Token
    @param salePrice ราคาขาย
    @return receiver ผู้รับ Royalty, royaltyAmount จำนวน Royalty
    """
    royalty: RoyaltyInfo = self.tokenRoyalties[tokenId]
    
    # ถ้าไม่มี Token-specific Royalty ใช้ Default
    if royalty.receiver == empty(address):
        royalty = self.defaultRoyalty
    
    royaltyAmount: uint256 = salePrice * royalty.royaltyBps / 10000
    return royalty.receiver, royaltyAmount

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return (
        interfaceId == ERC165_INTERFACE_ID or
        interfaceId == ERC721_INTERFACE_ID or
        interfaceId == ERC2981_INTERFACE_ID
    )

# ==================== Royalty Management ====================

@external
def setDefaultRoyalty(receiver: address, royaltyBps: uint256):
    """Set Default Royalty สำหรับทุก Token"""
    assert msg.sender == self.owner, "Not owner"
    assert royaltyBps <= MAX_ROYALTY_BPS, "Royalty too high"
    assert receiver != empty(address), "Zero receiver"
    
    self.defaultRoyalty = RoyaltyInfo({
        receiver: receiver,
        royaltyBps: royaltyBps
    })

@external
def setTokenRoyalty(tokenId: uint256, receiver: address, royaltyBps: uint256):
    """Set Royalty เฉพาะสำหรับ Token"""
    assert msg.sender == self.owner, "Not owner"
    assert royaltyBps <= MAX_ROYALTY_BPS, "Royalty too high"
    assert self.tokenOwners[tokenId] != empty(address), "Token does not exist"
    
    self.tokenRoyalties[tokenId] = RoyaltyInfo({
        receiver: receiver,
        royaltyBps: royaltyBps
    })
    
    log RoyaltySet(tokenId, receiver, royaltyBps)

@external
def resetTokenRoyalty(tokenId: uint256):
    """ล้าง Token-specific Royalty (กลับไปใช้ Default)"""
    assert msg.sender == self.owner, "Not owner"
    self.tokenRoyalties[tokenId] = RoyaltyInfo({
        receiver: empty(address),
        royaltyBps: 0
    })

# ==================== Core ERC-721 ====================

@external
@view
def balanceOf(_owner: address) -> uint256:
    return self.ownerTokenCount[_owner]

@external
@view
def ownerOf(tokenId: uint256) -> address:
    owner_addr: address = self.tokenOwners[tokenId]
    assert owner_addr != empty(address), "Token does not exist"
    return owner_addr

@external
def mint(to: address) -> uint256:
    assert msg.sender == self.owner, "Not owner"
    assert to != empty(address), "Zero address"
    
    tokenId: uint256 = self.nextTokenId
    self.nextTokenId += 1
    
    self.tokenOwners[tokenId] = to
    self.ownerTokenCount[to] += 1
    self.totalSupply += 1
    
    log Transfer(empty(address), to, tokenId)
    return tokenId

@external
def transferFrom(from_: address, to: address, tokenId: uint256):
    assert self.tokenOwners[tokenId] == from_, "Not owner"
    assert to != empty(address), "Zero address"
    assert (
        msg.sender == from_ or
        msg.sender == self.tokenOwners[tokenId]
    ), "Not authorized"
    
    self.ownerTokenCount[from_] -= 1
    self.ownerTokenCount[to] += 1
    self.tokenOwners[tokenId] = to
    
    log Transfer(from_, to, tokenId)
```

---

## 4. Complete Test Suite {#test-suite}

```python
# tests/test_erc721.py
"""
Complete Test Suite สำหรับ ERC-721 NFT
"""
import pytest
import boa

@pytest.fixture
def deployer():
    return boa.env.eoa

@pytest.fixture
def user1():
    return boa.env.generate_address()

@pytest.fixture
def user2():
    return boa.env.generate_address()

@pytest.fixture
def nft(deployer):
    return boa.load(
        "contracts/ERC721Full.vy",
        "Test NFT",
        "TNFT",
        "ipfs://QmBase/"
    )

class TestDeployment:
    def test_initial_state(self, nft, deployer):
        assert nft.name() == "Test NFT"
        assert nft.symbol() == "TNFT"
        assert nft.totalSupply() == 0
        assert nft.owner() == deployer

class TestMint:
    def test_mint(self, nft, deployer, user1):
        tokenId = nft.mint(user1, "ipfs://QmToken1/")
        
        assert nft.ownerOf(tokenId) == user1
        assert nft.balanceOf(user1) == 1
        assert nft.totalSupply() == 1

    def test_mint_increments_token_id(self, nft, user1):
        id1 = nft.mint(user1, "")
        id2 = nft.mint(user1, "")
        
        assert id2 == id1 + 1

    def test_non_minter_cannot_mint(self, nft, user1):
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="Not minter"):
                nft.mint(user1, "")

    def test_mint_emits_transfer_event(self, nft, user1):
        nft.mint(user1, "")
        
        logs = nft.get_logs()
        assert len(logs) > 0

class TestTransfer:
    def test_transfer_from(self, nft, deployer, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.transferFrom(user1, user2, tokenId)
        
        assert nft.ownerOf(tokenId) == user2
        assert nft.balanceOf(user1) == 0
        assert nft.balanceOf(user2) == 1

    def test_safe_transfer_to_eoa(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.safeTransferFrom(user1, user2, tokenId, b"")
        
        assert nft.ownerOf(tokenId) == user2

    def test_unauthorized_transfer_fails(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user2):
            with pytest.raises(Exception, match="Not authorized"):
                nft.transferFrom(user1, user2, tokenId)

    def test_transfer_nonexistent_fails(self, nft, user1, user2):
        with boa.env.prank(user1):
            with pytest.raises(Exception):
                nft.transferFrom(user1, user2, 9999)

class TestApproval:
    def test_approve(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.approve(user2, tokenId)
        
        assert nft.getApproved(tokenId) == user2

    def test_approved_can_transfer(self, nft, user1, user2):
        deployer = boa.env.generate_address()
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.approve(user2, tokenId)
        
        with boa.env.prank(user2):
            nft.transferFrom(user1, deployer, tokenId)
        
        assert nft.ownerOf(tokenId) == deployer

    def test_approval_cleared_after_transfer(self, nft, user1, user2):
        dest = boa.env.generate_address()
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.approve(user2, tokenId)
        
        with boa.env.prank(user2):
            nft.transferFrom(user1, dest, tokenId)
        
        assert nft.getApproved(tokenId) == "0x0000000000000000000000000000000000000000"

    def test_set_approval_for_all(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.setApprovalForAll(user2, True)
        
        assert nft.isApprovedForAll(user1, user2) == True
        
        dest = boa.env.generate_address()
        with boa.env.prank(user2):
            nft.transferFrom(user1, dest, tokenId)
        
        assert nft.ownerOf(tokenId) == dest

class TestBurn:
    def test_owner_can_burn(self, nft, user1):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.burn(tokenId)
        
        assert nft.totalSupply() == 0
        
        with pytest.raises(Exception):
            nft.ownerOf(tokenId)

    def test_approved_can_burn(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        
        with boa.env.prank(user1):
            nft.approve(user2, tokenId)
        
        with boa.env.prank(user2):
            nft.burn(tokenId)
        
        assert nft.totalSupply() == 0

class TestMetadata:
    def test_token_uri_with_base(self, nft, user1):
        tokenId = nft.mint(user1, "")
        uri = nft.tokenURI(tokenId)
        assert uri == f"ipfs://QmBase/{tokenId}"

    def test_token_uri_individual(self, nft, user1):
        tokenId = nft.mint(user1, "ipfs://QmSpecific/1")
        uri = nft.tokenURI(tokenId)
        assert uri == "ipfs://QmSpecific/1"

class TestRoyalties:
    @pytest.fixture
    def nft_with_royalty(self, deployer):
        creator = boa.env.generate_address()
        return boa.load(
            "contracts/ERC721Royalty.vy",
            "Royalty NFT",
            "RNFT",
            creator,
            500  # 5% royalty
        ), creator

    def test_royalty_info(self, nft_with_royalty):
        nft, creator = nft_with_royalty
        
        tokenId = nft.mint(boa.env.eoa)
        
        receiver, amount = nft.royaltyInfo(tokenId, 10**18)  # 1 ETH
        
        assert receiver == creator
        assert amount == 5 * 10**16  # 5% of 1 ETH = 0.05 ETH

    def test_supports_eip2981(self, nft_with_royalty):
        nft, _ = nft_with_royalty
        
        ERC2981_INTERFACE_ID = bytes.fromhex("2a55205a")
        assert nft.supportsInterface(ERC2981_INTERFACE_ID) == True

class TestERC165:
    def test_supports_erc721(self, nft):
        ERC721_ID = bytes.fromhex("80ac58cd")
        assert nft.supportsInterface(ERC721_ID) == True

    def test_supports_erc165(self, nft):
        ERC165_ID = bytes.fromhex("01ffc9a7")
        assert nft.supportsInterface(ERC165_ID) == True

    def test_not_support_random_interface(self, nft):
        RANDOM_ID = bytes.fromhex("deadbeef")
        assert nft.supportsInterface(RANDOM_ID) == False

class TestPause:
    def test_pause_blocks_transfer(self, nft, user1, user2):
        tokenId = nft.mint(user1, "")
        nft.pause()
        
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="Contract paused"):
                nft.transferFrom(user1, user2, tokenId)

class TestIntegration:
    def test_full_nft_lifecycle(self, nft, user1, user2):
        """ทดสอบ Life Cycle ทั้งหมด"""
        # Mint
        tokenId = nft.mint(user1, "ipfs://QmTest/1")
        assert nft.ownerOf(tokenId) == user1
        
        # Approve
        with boa.env.prank(user1):
            nft.approve(user2, tokenId)
        
        # Transfer via Approved
        dest = boa.env.generate_address()
        with boa.env.prank(user2):
            nft.transferFrom(user1, dest, tokenId)
        
        assert nft.ownerOf(tokenId) == dest
        assert nft.balanceOf(user1) == 0
        
        # Burn
        with boa.env.prank(dest):
            nft.burn(tokenId)
        
        assert nft.totalSupply() == 0
```

---

## 5. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Generative NFT
สร้าง NFT ที่ Generate Metadata On-chain:
- Random Traits จาก Block Hash
- SVG Image On-chain
- Rariry Tiers

### แบบฝึกหัดที่ 2: NFT Auction
สร้าง English Auction สำหรับ NFT:
- เพิ่ม Bid ได้ตลอด
- Auto-extend เมื่อมี Bid ท้ายๆ
- Royalties ไปยัง Creator

### แบบฝึกหัดที่ 3: Soulbound Token
สร้าง Non-transferable NFT:
- ออก Certification
- ไม่สามารถโอนได้
- สามารถ Revoke ได้โดย Issuer

---

## สรุป

| Feature | Implementation |
|---------|---------------|
| ERC-721 Core | ✅ ครบถ้วน |
| Metadata Extension | ✅ tokenURI |
| Enumerable Extension | ✅ tokenByIndex, tokensOfOwner |
| Royalties (EIP-2981) | ✅ Per-token Royalty |
| ERC-165 | ✅ supportsInterface |
| Pause/Unpause | ✅ Emergency Stop |
| Burn | ✅ Token Destruction |
| Test Coverage | ✅ ครอบคลุม |

---

[← Part 023: ERC-721 Standard](part_023_erc721_standard.md) | [Part 025: ERC-1155 →](part_025_erc1155.md)
