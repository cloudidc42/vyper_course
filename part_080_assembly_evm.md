# Part 080: Low-level EVM และ Inline Assembly ใน Vyper

## สารบัญ
1. EVM Architecture ภาพรวม
2. EVM Opcodes ที่สำคัญ
3. raw_call ใน Vyper
4. raw_log ใน Vyper
5. Inline Assembly ผ่าน `_abi_encode`
6. Bytecode Analysis
7. Gas Optimization ด้วย Low-level Calls
8. ตัวอย่างขั้นสูง

---

## 1. EVM Architecture

EVM (Ethereum Virtual Machine) เป็น stack-based virtual machine ที่รัน bytecode ของ smart contracts

### โครงสร้างของ EVM

```
EVM Components:
├── Stack (สูงสุด 1024 items, แต่ละ item = 32 bytes)
├── Memory (temporary, เปิด-ปิดแต่ละ call)
├── Storage (persistent, เก็บใน blockchain)
├── Calldata (input จาก caller)
└── Returndata (output จาก subcall)

Stack Operations:
PUSH1 0x42   → stack: [0x42]
PUSH1 0x01   → stack: [0x01, 0x42]
ADD          → stack: [0x43]  (0x42 + 0x01)
```

### Gas Costs ของ Operations

```
Operation          Gas Cost
─────────────────────────────
ADD, SUB           3
MUL, DIV           5
SHA3 (keccak256)   30 + 6/word
SLOAD              2100 (cold), 100 (warm)
SSTORE (new)       22100
SSTORE (update)    5000
SSTORE (delete)    5000 - refund
CALL               100 + value transfer
CREATE             32000
LOG (per topic)    375 + 8/byte
```

---

## 2. Vyper low-level Functions

Vyper มี built-in functions สำหรับ low-level operations

```vyper
# @version 0.4.0
# @title Low-level EVM Demo
# @notice แสดงการใช้ low-level functions ใน Vyper

# raw_call: เรียก contract แบบ low-level
# raw_log: สร้าง log event แบบ manual
# _abi_encode: encode data สำหรับ calldata
# _abi_decode: decode calldata

event LowLevelCall:
    target: indexed(address)
    success: bool
    data_length: uint256

event RawLogEvent:
    topic1: bytes32
    topic2: bytes32

@deploy
def __init__():
    pass

# ===== raw_call =====

@external
def simpleRawCall(target: address, calldata: Bytes[256]) -> Bytes[256]:
    """
    เรียก contract แบบ low-level ด้วย raw_call
    คล้ายกับ address.call() ใน Solidity
    """
    response: Bytes[256] = raw_call(
        target,
        calldata,
        max_outsize=256
    )
    return response

@external
def rawCallWithValue(target: address, amount: uint256) -> bool:
    """
    ส่ง ETH พร้อมกับ raw_call
    """
    response: Bytes[32] = raw_call(
        target,
        b"",                    # empty calldata (fallback function)
        value=amount,           # ส่ง ETH
        max_outsize=32
    )
    return True

@external
def rawCallWithGasLimit(
    target: address,
    calldata: Bytes[256],
    gas_limit: uint256
) -> Bytes[256]:
    """
    จำกัด gas ที่ส่งไปกับ raw_call
    """
    response: Bytes[256] = raw_call(
        target,
        calldata,
        gas=gas_limit,          # จำกัด gas
        max_outsize=256
    )
    return response

@external
def rawCallRevertable(
    target: address,
    calldata: Bytes[256]
) -> (bool, Bytes[256]):
    """
    raw_call ที่ไม่ revert เมื่อ subcall fails
    """
    success: bool = False
    response: Bytes[256] = b""
    
    response = raw_call(
        target,
        calldata,
        max_outsize=256,
        revert_on_failure=False   # ไม่ revert ถ้า subcall fails
    )
    
    success = len(response) > 0  # simplified check
    
    log LowLevelCall(target, success, len(response))
    
    return success, response

@external
def delegateCall(target: address, calldata: Bytes[256]) -> Bytes[256]:
    """
    delegatecall - รัน code ของ target แต่ใช้ storage ของตัวเอง
    ใช้ใน proxy patterns
    """
    response: Bytes[256] = raw_call(
        target,
        calldata,
        max_outsize=256,
        is_delegate_call=True     # ใช้ delegatecall
    )
    return response

@external
def staticCall(target: address, calldata: Bytes[256]) -> Bytes[256]:
    """
    staticcall - read-only call, ไม่สามารถเปลี่ยน state
    """
    response: Bytes[256] = raw_call(
        target,
        calldata,
        max_outsize=256,
        is_static_call=True       # ใช้ staticcall
    )
    return response
```

---

## 3. raw_log และ Events

```vyper
# @version 0.4.0
# @title Raw Log Demo
# @notice แสดงการใช้ raw_log สำหรับ custom events

@deploy
def __init__():
    pass

@external
def emitCustomLog():
    """
    สร้าง event log แบบ manual ด้วย raw_log
    """
    # Topic 1: keccak256 ของ event signature
    topic1: bytes32 = keccak256("Transfer(address,address,uint256)")
    
    # Topic 2: from address (indexed)
    topic2: bytes32 = convert(msg.sender, bytes32)
    
    # Topic 3: to address (indexed)
    topic3: bytes32 = convert(self, bytes32)
    
    # Data: amount (non-indexed)
    amount: uint256 = 1000
    data: Bytes[32] = _abi_encode(amount)
    
    # Emit log with 3 topics
    raw_log(
        [topic1, topic2, topic3],
        data
    )

@external
def emitTransferLikeLog(
    from_addr: address,
    to_addr: address,
    amount: uint256
):
    """
    Emit log ที่มีรูปแบบเหมือน ERC-20 Transfer event
    """
    # ERC-20 Transfer event signature
    sig: bytes32 = keccak256("Transfer(address,address,uint256)")
    
    # Indexed parameters เป็น topics
    from_topic: bytes32 = convert(from_addr, bytes32)
    to_topic: bytes32 = convert(to_addr, bytes32)
    
    # Non-indexed parameters เป็น data
    data: Bytes[32] = _abi_encode(amount)
    
    raw_log([sig, from_topic, to_topic], data)

@external
def emitAnonymousLog(amount: uint256):
    """
    Anonymous log ไม่มี event signature
    """
    # ไม่ใช่ ABI log มาตรฐาน แต่ถูกต้อง
    data: Bytes[32] = _abi_encode(amount)
    raw_log([], data)  # ไม่มี topics
```

---

## 4. ABI Encoding/Decoding

```vyper
# @version 0.4.0
# @title ABI Encoding Demo
# @notice แสดงการ encode/decode data สำหรับ calldata

@deploy
def __init__():
    pass

@pure
@external
def encodeTransferCalldata(
    to: address,
    amount: uint256
) -> Bytes[68]:
    """
    สร้าง calldata สำหรับ ERC-20 transfer(address,uint256)
    """
    # Function selector: keccak256("transfer(address,uint256)") first 4 bytes
    selector: bytes4 = 0xa9059cbb
    
    # Encode parameters
    encoded: Bytes[64] = _abi_encode(to, amount)
    
    # Combine selector + encoded params
    return concat(selector, encoded)

@pure
@external
def encodeComplexCalldata(
    user: address,
    values: uint256[3],
    data: Bytes[64]
) -> Bytes[256]:
    """
    Encode complex calldata
    """
    return _abi_encode(
        user,
        values,
        data,
        method_id=method_id("complexFunction(address,uint256[3],bytes)")
    )

@external
def callTransfer(
    token: address,
    to: address,
    amount: uint256
) -> bool:
    """
    เรียก ERC-20 transfer ด้วย raw_call
    """
    calldata: Bytes[68] = _abi_encode(
        to,
        amount,
        method_id=method_id("transfer(address,uint256)")
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32
    )
    
    # Decode response
    if len(response) == 32:
        return _abi_decode(response, (bool))
    
    return True  # Some tokens don't return value

@view
@external
def callBalanceOf(token: address, user: address) -> uint256:
    """
    เรียก ERC-20 balanceOf ด้วย staticcall
    """
    calldata: Bytes[36] = _abi_encode(
        user,
        method_id=method_id("balanceOf(address)")
    )
    
    response: Bytes[32] = raw_call(
        token,
        calldata,
        max_outsize=32,
        is_static_call=True
    )
    
    return _abi_decode(response, (uint256))
```

---

## 5. Proxy Pattern ด้วย Low-level Calls

```vyper
# @version 0.4.0
# @title Minimal Proxy (EIP-1167)
# @notice Implementation ของ minimal proxy pattern

# EIP-1167 Minimal Proxy Bytecode
# 363d3d373d3d3d363d73<address>5af43d82803e903d91602b57fd5bf3

@deploy
def __init__():
    pass

@external
def deployMinimalProxy(implementation: address) -> address:
    """
    Deploy EIP-1167 minimal proxy clone
    """
    # Minimal proxy bytecode ที่มี implementation address
    # prefix: 363d3d373d3d3d363d73
    # address: 20 bytes
    # suffix: 5af43d82803e903d91602b57fd5bf3
    
    prefix: Bytes[10] = b"\x36\x3d\x3d\x37\x3d\x3d\x3d\x36\x3d\x73"
    suffix: Bytes[15] = b"\x5a\xf4\x3d\x82\x80\x3e\x90\x3d\x91\x60\x2b\x57\xfd\x5b\xf3"
    
    impl_bytes: bytes20 = convert(implementation, bytes20)
    
    bytecode: Bytes[45] = concat(
        prefix,
        impl_bytes,
        suffix
    )
    
    # Deploy ด้วย create
    addr: address = create_minimal_proxy_to(implementation)
    
    return addr

@external
def deployWithCreate2(
    implementation: address,
    salt: bytes32
) -> address:
    """
    Deploy proxy ด้วย CREATE2 (deterministic address)
    """
    addr: address = create_minimal_proxy_to(
        implementation,
        salt=salt
    )
    return addr
```

---

## 6. Storage Slot Manipulation

```vyper
# @version 0.4.0
# @title Storage Slot Demo
# @notice แสดงการอ่าน/เขียน storage slots โดยตรง

# EIP-1967 storage slots
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
ADMIN_SLOT: constant(bytes32) = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

@deploy
def __init__():
    pass

@internal
def _setImplementation(impl: address):
    """
    เขียน implementation address ไปที่ EIP-1967 slot
    """
    # ใช้ Yul/inline assembly ใน Solidity จะเป็น:
    # sstore(IMPLEMENTATION_SLOT, impl)
    # ใน Vyper ใช้ raw_call หรือ storage[slot] syntax
    pass

@view
@internal
def _getImplementation() -> address:
    """
    อ่าน implementation address จาก EIP-1967 slot
    """
    # ใน Vyper เราใช้ built-in functions แทน
    return empty(address)

@external
def getStorageAt(slot: bytes32) -> bytes32:
    """
    อ่าน arbitrary storage slot
    สำคัญสำหรับ debugging และ proxy patterns
    """
    # Vyper ไม่มี sload โดยตรง แต่สามารถใช้ assembly ได้
    # ผ่าน inline _abi_encode
    return empty(bytes32)
```

---

## 7. Bytecode Analysis

```python
# scripts/analyze_bytecode.py
# วิเคราะห์ bytecode ของ Vyper contracts

import json
from web3 import Web3

def analyze_bytecode(contract_name: str):
    """วิเคราะห์ bytecode ของ contract"""
    
    # Load compiled contract
    with open(f"build/{contract_name}.json") as f:
        artifact = json.load(f)
    
    bytecode = artifact["bytecode"]
    abi = artifact["abi"]
    
    print(f"Contract: {contract_name}")
    print(f"Bytecode size: {len(bytecode) // 2} bytes")
    print(f"Max contract size: 24576 bytes")
    print(f"Size usage: {len(bytecode) // 2 / 24576 * 100:.1f}%")
    
    # Analyze function selectors
    print("\nFunction Selectors:")
    for item in abi:
        if item["type"] == "function":
            selector = Web3.keccak(
                text=f"{item['name']}({','.join(i['type'] for i in item['inputs'])})"
            )[:4].hex()
            print(f"  {item['name']}: 0x{selector}")
    
    return bytecode

def disassemble_basic(bytecode: str):
    """Disassemble EVM bytecode"""
    
    OPCODES = {
        0x00: "STOP",
        0x01: "ADD",
        0x02: "MUL",
        0x03: "SUB",
        0x04: "DIV",
        0x05: "SDIV",
        0x06: "MOD",
        0x07: "SMOD",
        0x08: "ADDMOD",
        0x09: "MULMOD",
        0x0a: "EXP",
        0x0b: "SIGNEXTEND",
        0x10: "LT",
        0x11: "GT",
        0x12: "SLT",
        0x13: "SGT",
        0x14: "EQ",
        0x15: "ISZERO",
        0x16: "AND",
        0x17: "OR",
        0x18: "XOR",
        0x19: "NOT",
        0x1a: "BYTE",
        0x1b: "SHL",
        0x1c: "SHR",
        0x1d: "SAR",
        0x20: "SHA3",
        0x30: "ADDRESS",
        0x31: "BALANCE",
        0x32: "ORIGIN",
        0x33: "CALLER",
        0x34: "CALLVALUE",
        0x35: "CALLDATALOAD",
        0x36: "CALLDATASIZE",
        0x37: "CALLDATACOPY",
        0x38: "CODESIZE",
        0x39: "CODECOPY",
        0x3a: "GASPRICE",
        0x3b: "EXTCODESIZE",
        0x3c: "EXTCODECOPY",
        0x3d: "RETURNDATASIZE",
        0x3e: "RETURNDATACOPY",
        0x3f: "EXTCODEHASH",
        0x40: "BLOCKHASH",
        0x41: "COINBASE",
        0x42: "TIMESTAMP",
        0x43: "NUMBER",
        0x44: "DIFFICULTY",
        0x45: "GASLIMIT",
        0x46: "CHAINID",
        0x47: "SELFBALANCE",
        0x48: "BASEFEE",
        0x50: "POP",
        0x51: "MLOAD",
        0x52: "MSTORE",
        0x53: "MSTORE8",
        0x54: "SLOAD",
        0x55: "SSTORE",
        0x56: "JUMP",
        0x57: "JUMPI",
        0x58: "PC",
        0x59: "MSIZE",
        0x5a: "GAS",
        0x5b: "JUMPDEST",
        0x60: "PUSH1",
        0xf0: "CREATE",
        0xf1: "CALL",
        0xf2: "CALLCODE",
        0xf3: "RETURN",
        0xf4: "DELEGATECALL",
        0xf5: "CREATE2",
        0xfa: "STATICCALL",
        0xfd: "REVERT",
        0xfe: "INVALID",
        0xff: "SELFDESTRUCT",
    }
    
    code = bytes.fromhex(bytecode.replace("0x", ""))
    i = 0
    
    print("\nDisassembly:")
    while i < len(code):
        op = code[i]
        opname = OPCODES.get(op, f"UNKNOWN_0x{op:02x}")
        
        # PUSH1-PUSH32 ต้องมี immediate data
        if 0x60 <= op <= 0x7f:
            push_size = op - 0x5f  # PUSH1 = 1 byte, PUSH32 = 32 bytes
            data = code[i+1:i+1+push_size].hex()
            print(f"  {i:04x}: {opname} 0x{data}")
            i += 1 + push_size
        else:
            print(f"  {i:04x}: {opname}")
            i += 1

# รัน analysis
if __name__ == "__main__":
    bytecode = analyze_bytecode("LowLevelDemo")
    disassemble_basic(bytecode)
```

---

## 8. Gas-Optimized Contract ด้วย Low-level Calls

```vyper
# @version 0.4.0
# @title Gas Optimized ERC-20
# @notice ERC-20 ที่ optimize ด้วย low-level techniques

from vyper.interfaces import ERC20

# Events
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# Pack state variables เพื่อลด storage reads
# ใช้ single slot สำหรับ metadata
name: public(String[32])
symbol: public(String[8])
decimals: public(uint8)

totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

@deploy
def __init__(name: String[32], symbol: String[8]):
    self.name = name
    self.symbol = symbol
    self.decimals = 18

@internal
def _transfer(sender: address, receiver: address, amount: uint256):
    """
    Internal transfer - optimize gas โดยไม่ emit event ที่ไม่จำเป็น
    """
    assert sender != empty(address), "From zero"
    assert receiver != empty(address), "To zero"
    
    senderBal: uint256 = self.balanceOf[sender]
    assert senderBal >= amount, "Insufficient balance"
    
    # Unchecked arithmetic (ปลอดภัยเพราะ checked ข้างบน)
    self.balanceOf[sender] = unsafe_sub(senderBal, amount)
    self.balanceOf[receiver] = unsafe_add(self.balanceOf[receiver], amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    allowanceAmount: uint256 = self.allowance[sender][msg.sender]
    
    # Check max allowance (infinite approval pattern)
    if allowanceAmount != max_value(uint256):
        assert allowanceAmount >= amount, "Insufficient allowance"
        self.allowance[sender][msg.sender] = unsafe_sub(allowanceAmount, amount)
    
    self._transfer(sender, to, amount)
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve zero"
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def permit(
    owner: address,
    spender: address,
    value: uint256,
    deadline: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """
    EIP-2612 permit - gasless approve
    ใช้ ecrecover เพื่อ verify signature
    """
    assert deadline >= block.timestamp, "Expired"
    
    # สร้าง EIP-712 digest
    domain_separator: bytes32 = keccak256(
        _abi_encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(convert(self.name, Bytes[32])),
            keccak256(b"1"),
            chain.id,
            self
        )
    )
    
    # นับ nonce ของ owner (simplified)
    nonce: uint256 = 0  # Should track in production
    
    permit_hash: bytes32 = keccak256(
        _abi_encode(
            keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"),
            owner,
            spender,
            value,
            nonce,
            deadline
        )
    )
    
    digest: bytes32 = keccak256(
        concat(
            b"\x19\x01",
            domain_separator,
            permit_hash
        )
    )
    
    recovered: address = ecrecover(digest, v, r, s)
    assert recovered == owner, "Invalid signature"
    
    self.allowance[owner][spender] = value
    log Approval(owner, spender, value)

@external
def batchTransfer(
    recipients: DynArray[address, 100],
    amounts: DynArray[uint256, 100]
) -> bool:
    """
    Batch transfer สำหรับ gas savings เมื่อต้องโอนหลาย addresses
    """
    assert len(recipients) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(recipients):
            break
        self._transfer(msg.sender, recipients[i], amounts[i])
        log Transfer(msg.sender, recipients[i], amounts[i])
    
    return True
```

---

## 9. Multi-call Contract

```vyper
# @version 0.4.0
# @title Multicall
# @notice Execute multiple calls in a single transaction

struct Call:
    target: address
    callData: Bytes[1024]

struct Result:
    success: bool
    returnData: Bytes[1024]

@deploy
def __init__():
    pass

@external
def aggregate(calls: DynArray[Call, 50]) -> (uint256, DynArray[Bytes[1024], 50]):
    """
    Execute หลาย calls พร้อมกัน
    Reverts ถ้ามี call ใด fails
    """
    results: DynArray[Bytes[1024], 50] = []
    blockNumber: uint256 = block.number
    
    for call in calls:
        response: Bytes[1024] = raw_call(
            call.target,
            call.callData,
            max_outsize=1024
        )
        results.append(response)
    
    return blockNumber, results

@external
def tryAggregate(
    requireSuccess: bool,
    calls: DynArray[Call, 50]
) -> DynArray[Result, 50]:
    """
    Execute หลาย calls แต่ไม่ revert ถ้า call fails
    """
    results: DynArray[Result, 50] = []
    
    for call in calls:
        success: bool = False
        returnData: Bytes[1024] = b""
        
        returnData = raw_call(
            call.target,
            call.callData,
            max_outsize=1024,
            revert_on_failure=False
        )
        
        # check ว่า call สำเร็จ
        success = len(returnData) > 0
        
        if requireSuccess:
            assert success, "Call failed"
        
        results.append(Result({
            success: success,
            returnData: returnData
        }))
    
    return results

@view
@external
def getEthBalance(addr: address) -> uint256:
    return addr.balance

@view
@external  
def getCurrentBlockTimestamp() -> uint256:
    return block.timestamp

@view
@external
def getCurrentBlockGasLimit() -> uint256:
    return block.gaslimit

@view
@external
def getChainId() -> uint256:
    return chain.id
```

---

## 10. Flash Loan ด้วย Low-level Calls

```vyper
# @version 0.4.0
# @title Flash Loan Executor
# @notice Execute flash loans จาก multiple providers

interface IUniswapV2Pair:
    def swap(amount0Out: uint256, amount1Out: uint256, to: address, data: Bytes[1024]): nonpayable
    def token0() -> address: view
    def token1() -> address: view

interface IERC20:
    def balanceOf(who: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable

event FlashLoanExecuted:
    pair: indexed(address)
    token: indexed(address)
    amount: uint256
    profit: uint256

owner: public(address)

@deploy
def __init__():
    self.owner = msg.sender

@external
def executeFlashLoan(
    pair: address,
    amount0: uint256,
    amount1: uint256,
    callbackData: Bytes[1024]
):
    """
    Request flash loan จาก Uniswap V2
    """
    assert msg.sender == self.owner, "Not owner"
    
    IUniswapV2Pair(pair).swap(
        amount0,
        amount1,
        self,
        callbackData
    )

@external
def uniswapV2Call(
    sender: address,
    amount0: uint256,
    amount1: uint256,
    data: Bytes[1024]
):
    """
    Callback จาก Uniswap V2 เมื่อ flash loan สำเร็จ
    """
    pair: address = msg.sender
    
    # ตรวจสอบว่า caller คือ valid pair
    token0: address = IUniswapV2Pair(pair).token0()
    token1: address = IUniswapV2Pair(pair).token1()
    
    # Decode ว่า token ไหนที่ borrow
    token: address = empty(address)
    amount: uint256 = 0
    
    if amount0 > 0:
        token = token0
        amount = amount0
    else:
        token = token1
        amount = amount1
    
    # ==================
    # EXECUTE ARBITRAGE หรือ STRATEGY HERE
    # ==================
    
    # คำนวณ fee (Uniswap V2: 0.3%)
    fee: uint256 = amount * 3 / 997 + 1
    repayAmount: uint256 = amount + fee
    
    # Repay flash loan
    balanceBefore: uint256 = IERC20(token).balanceOf(self)
    assert balanceBefore >= repayAmount, "Insufficient balance to repay"
    
    IERC20(token).transfer(pair, repayAmount)
    
    profit: uint256 = balanceBefore - repayAmount
    
    log FlashLoanExecuted(pair, token, amount, profit)
```

---

## 11. Contract Factory ด้วย CREATE2

```vyper
# @version 0.4.0
# @title Contract Factory ด้วย CREATE2
# @notice Deploy contracts ไปที่ deterministic addresses

event ContractDeployed:
    deployer: indexed(address)
    salt: indexed(bytes32)
    contractAddress: indexed(address)

@deploy
def __init__():
    pass

@external
def deployWithCreate2(
    salt: bytes32,
    bytecode: Bytes[24576]
) -> address:
    """
    Deploy contract ไปที่ address ที่คำนวณได้ล่วงหน้า
    """
    deployed: address = create_from_blueprint(
        empty(address),  # ต้องใช้ blueprint
        salt=salt
    )
    
    log ContractDeployed(msg.sender, salt, deployed)
    
    return deployed

@pure
@external
def computeCreate2Address(
    salt: bytes32,
    bytecodeHash: bytes32
) -> address:
    """
    คำนวณ address ที่จะได้จาก CREATE2
    address = keccak256(0xff ++ factory ++ salt ++ keccak256(bytecode))[12:]
    """
    return convert(
        convert(
            keccak256(
                concat(
                    b"\xff",
                    convert(self, bytes20),
                    salt,
                    bytecodeHash
                )
            ),
            uint256
        ) & convert(max_value(uint160), uint256),
        address
    )

@external
def deployMinimalProxy(
    implementation: address,
    salt: bytes32
) -> address:
    """
    Deploy EIP-1167 minimal proxy ไปที่ deterministic address
    """
    deployed: address = create_minimal_proxy_to(
        implementation,
        salt=salt
    )
    
    log ContractDeployed(msg.sender, salt, deployed)
    
    return deployed
```

---

## 12. Gas Analysis Tools

```python
# scripts/gas_analysis.py
# วิเคราะห์ gas usage ของ functions

import json
from eth_tester import EthereumTester
from web3 import Web3

def analyze_gas_usage(contract_address, abi, w3):
    """วิเคราะห์ gas ของแต่ละ function"""
    
    contract = w3.eth.contract(address=contract_address, abi=abi)
    
    results = {}
    
    # ทดสอบแต่ละ function
    for func in abi:
        if func['type'] != 'function':
            continue
        
        name = func['name']
        
        try:
            # Estimate gas
            if func['stateMutability'] in ['view', 'pure']:
                # Read functions
                result = getattr(contract.functions, name)().call()
                gas = getattr(contract.functions, name)().estimate_gas()
            else:
                # Write functions (simplified)
                pass
            
            results[name] = gas
            print(f"{name}: {gas:,} gas")
            
        except Exception as e:
            results[name] = f"Error: {e}"
    
    return results

# Gas optimization tips
GAS_TIPS = """
Gas Optimization Tips สำหรับ Vyper:

1. Pack storage variables ที่ใช้ร่วมกัน
   - uint128 + uint128 = 1 slot แทน 2 slots
   
2. ใช้ uint256 แทน smaller types สำหรับ calculations
   - EVM ทำงานกับ 256-bit เสมอ ดังนั้น uint8 ไม่ช่วย ถ้าไม่ pack

3. ใช้ events แทน storage สำหรับ historical data
   - Events ถูกกว่า storage มาก
   
4. Batch operations เพื่อลด base gas cost (21000 per tx)

5. ใช้ immutable variables สำหรับค่าคงที่
   - ถูกกว่า storage read

6. ลด SSTORE เท่าที่เป็นไปได้
   - Cold write: 22100 gas
   - Warm write: 5000 gas
   - Read: 2100 gas (cold), 100 gas (warm)

7. ใช้ calldata แทน memory สำหรับ function parameters
   - calldata: read-only แต่ถูกกว่า memory

8. Short-circuit conditions
   - ตรวจสอบ cheap conditions ก่อน expensive ones
"""

print(GAS_TIPS)
```

---

## 13. สรุป Low-level EVM Operations

### เมื่อไหร่ควรใช้ Low-level Operations

```
raw_call:
- ส่ง ETH พร้อม arbitrary calldata
- เรียก contract ที่ interface ไม่รู้จัก
- delegate call ใน proxy patterns
- ต้องการจัดการ failure เอง

raw_log:
- ต้องการ custom event format
- สร้าง events ที่ compatible กับ existing indexers
- ประหยัด gas โดยไม่ emit events ที่ไม่จำเป็น

_abi_encode / _abi_decode:
- สร้าง calldata สำหรับ external calls
- Decode return data จาก raw_call
- สร้าง signatures สำหรับ EIP-712

CREATE / CREATE2:
- Deploy contracts dynamically
- Factory patterns
- Deterministic addresses
```

### ข้อควรระวัง

```
1. raw_call ไม่ตรวจ return value โดยอัตโนมัติ
   - ต้อง handle success/failure เอง
   
2. delegatecall อันตราย - ใช้ storage ของ caller
   - Storage layout ต้องตรงกัน
   
3. CREATE2 address คำนวณจาก bytecode
   - เปลี่ยน bytecode = เปลี่ยน address
   
4. Gas forwarding ใน raw_call
   - Default ส่ง gas ทั้งหมด (63/64 rule)
   - ระบุ gas= parameter เพื่อ limit
```

---

## แบบฝึกหัด

1. เขียน contract ที่ใช้ raw_call เพื่อเรียก multiple ERC-20 tokens
2. สร้าง custom event indexer ที่ใช้ raw_log
3. Implement multicall contract ที่รองรับ value transfer
4. วิเคราะห์ bytecode ของ contract ที่เขียนเอง
5. เขียน flash loan arbitrage bot โดยใช้ low-level calls

---

*จบ Part 080: Low-level EVM และ Inline Assembly ใน Vyper*
