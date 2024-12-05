## **VoucherToken 스마트 컨트랙트**

VoucherToken 스마트 컨트랙트는 탄소 상쇄권과 연동된 ERC-1155 기반 NFT 토큰을 발행, 관리 및 거래하기 위해 설계되었습니다. 이 컨트랙트는 다음과 같은 주요 기능을 제공합니다.

1. **운영자 기반 민팅**
2. **서명 기반 민팅 및 전송**
3. **수수료 처리**

------

### **함수 목록 및 설명**

#### **1. 생성자**

**설명:** 컨트랙트를 초기화하며 토큰의 기본 정보와 수수료 관리 컨트랙트 주소를 설정합니다.

**파라미터**

- `_usdtContractAddress` (address) : USDT (Tether) 토큰의 스마트 계약 주소
- `_operatorManager` (address) : 운영자 권한을 관리하는 스마트 계약의 주소
- `_whitelistManager` (address) : 화이트리스트 기능을 관리하는 스마트 계약의 주소
- `_feeManager` (address) : 수수료 정책을 관리하는 스마트 계약의 주소

**로직**

1. 전달된 `_usdtContractAddress`, `_operatorManager`, `_whitelistManager`, `_feeManager` 값을 각각의 상태 변수에 저장합니다.

```solidity
constructor(
    address _usdtContractAddress,
    address _operatorManager,
    address _whitelistManager,
    address _feeManager
) {
    usdtContractAddress = _usdtContractAddress;
    operatorManager = _operatorManager;
    whitelistManager = _whitelistManager;
    feeManager = _feeManager;
}
```

------

#### **2. `mintByOperator`**

**설명:** 운영자가 토큰을 민팅할 수 있도록 허용합니다.

 **파라미터:**

- `_receiver` (address): 민팅된 토큰을 받을 주소
- `_amount` (uint256): 민팅할 토큰 수량
- `_tokenId` (uint256): 민팅할 토큰 ID
- `_metadata` (string): 토큰 메타데이터 URI

**로직:**

1. `_metadata`를 `tokenURIs`에 저장합니다.
2. `_amount`를 `tokenSupply[_tokenId]`에 추가합니다.
3. `_mint` 함수를 호출해 토큰을 발행합니다.

```solidity
function mintByOperator(address _receiver, uint256 _amount, uint256 _tokenId, string memory _metadata) external override onlyOwner {
    tokenURIs[_tokenId] = _metadata;
    tokenSupply[_tokenId] = tokenSupply[_tokenId].add(_amount);
    _mint(_receiver, _tokenId, _amount, "");
}
```

------

#### **3. `mintBySignature`**

**설명:** 서명 검증을 통해 토큰을 민팅합니다.

 **파라미터:**

- `_receiver` (address): 민팅된 토큰을 받을 주소
- `_amount` (uint256): 민팅할 토큰 수량
- `_tokenId` (uint256): 민팅할 토큰 ID
- `_nonce` (uint256): 재생 공격 방지용 값
- `_metadata` (string): 토큰 메타데이터 URI
- `signature` (bytes): 서명 데이터
- `_carbonPrice` (uint256): 토큰의 탄소 가격

**로직:**

1. `_carbonPrice`가 최소 가격보다 큰지 확인합니다.
2. 서명 데이터(`signature`)를 복호화하고, `_receiver`가 서명자인지 확인합니다.
3. 트랜잭션 해시를 생성하고, 중복 트랜잭션 여부를 검사합니다.
4. 수수료 계산 후, `_receiver`에게 남은 토큰 수량을 발행합니다.
5. 수수료는 `feeManager`가 설정한 주소로 전송됩니다.
6. `carbonMapPrice[_tokenId]`를 `_carbonPrice`로 업데이트합니다.

```solidity
function mintBySignature(
    address _receiver, uint256 _amount, uint256 _tokenId, uint256 _nonce,
    string memory _metadata, bytes memory signature, uint256 _carbonPrice
) external onlyOwner {
    require(_carbonPrice >= minUSDTPrice, "price must be higher than min");
    require(_carbonPrice > carbonMapPrice[_tokenId], "price must be higher than old");

    bytes32 hashMessage = keccak256(abi.encodePacked(_receiver, _tokenId, _amount, _nonce, address(this)));
    bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hashMessage));
    address signer = recoverSigner(hash, signature);

    require(signer == _receiver, "Signature does not match the sender");
    require(!transactionHashes[hashMessage], "Transaction already processed");

    transactionHashes[hashMessage] = true;

    uint256 calculatedAmount = IFeeManager(feeManager).feeAmount(_amount);
    uint256 remainAmount = _amount.sub(calculatedAmount);

    tokenURIs[_tokenId] = _metadata;
    tokenSupply[_tokenId] = tokenSupply[_tokenId].add(_amount);

    _mint(_receiver, _tokenId, remainAmount, "");
    _mint(IFeeManager(feeManager).feeAddress(), _tokenId, calculatedAmount, "");

    carbonMapPrice[_tokenId] = _carbonPrice;
}
```

------

#### **4. `transferBySignature`**

**설명:** 서명을 통해 ERC-1155 토큰을 전송합니다.

 **파라미터:**

- `from` (address): 전송할 토큰을 보유한 주소
- `to` (address): 토큰을 받을 주소
- `tokenId` (uint256): 전송할 토큰 ID
- `amount` (uint256): 전송할 토큰 수량
- `nonce` (uint256): 재생 공격 방지용 값
- `signature` (bytes): 서명 데이터

**로직:**

1. 서명을 통해 전송 요청을 검증합니다.
2. 트랜잭션 해시를 생성하고 중복 요청 여부를 확인합니다.
3. 수수료 계산 후, `to`에게 남은 토큰을 전송합니다.
4. 수수료는 `feeManager`가 설정한 주소로 전송됩니다.

```solidity
function transferBySignature(
    address from, address to, uint256 tokenId, uint256 amount, uint256 nonce, bytes memory signature
) external nonReentrant onlyOwner {
    bytes32 hashMessage = keccak256(abi.encodePacked(from, to, tokenId, amount, nonce, address(this)));
    bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hashMessage));
    address signer = recoverSigner(hash, signature);

    require(signer == from, "Signature does not match the sender");
    require(!transactionHashes[hashMessage], "Transaction already processed");

    transactionHashes[hashMessage] = true;

    uint256 calculatedAmount = IFeeManager(feeManager).feeAmount(amount);
    uint256 remainAmount = amount.sub(calculatedAmount);

    _safeTransferFrom(from, to, tokenId, remainAmount, "");
    _safeTransferFrom(from, IFeeManager(feeManager).feeAddress(), tokenId, calculatedAmount, "");
}
```

---

#### 5. `recoverSigner`

**설명:** `recoverSigner` 함수는 서명을 복구하여 서명자가 누구인지 확인하는 데 사용됩니다.
Ethereum에서 서명은 메시지의 무결성을 확인하고 특정 개인(private key)만이 해당 서명을 생성했음을 증명합니다.
이 함수는 ECDSA(Elliptic Curve Digital Signature Algorithm)를 사용하여 서명에서 서명자의 주소를 복구합니다.

**파라미터**

- `hash`  (bytes32): 서명된 데이터의 해시
- `signature` (bytes): 서명 데이터

**로직**

1. **서명 분리:**
   서명 데이터(`signature`)를 `r`, `s`, `v` 값으로 분리합니다.
   - `r`: 서명의 첫 번째 부분
   - `s`: 서명의 두 번째 부분
   - `v`: 복구 ID (서명자가 만든 두 개의 서명 중 하나를 식별)
2. **주소 복구:**
   `ecrecover`를 사용하여 서명에서 서명자 주소를 복구합니다.
   - `ecrecover`는 메시지 해시, `v`, `r`, `s`를 입력으로 받아 서명자의 주소를 반환합니다.
3. **주소 반환:**
   복구된 주소를 반환하여 서명이 유효한지 확인합니다.

```solidity
function recoverSigner(bytes32 hash, bytes memory signature) internal pure returns (address) {
    require(signature.length == 65, "Invalid signature length");

    bytes32 r;
    bytes32 s;
    uint8 v;

    (r, s, v) = splitSignature(signature);

    address signer = ecrecover(hash, v, r, s);
    require(signer != address(0), "Invalid signature");

    return signer;
}
```

------

## VoucherTokenMarket 스마트 컨트랙트

------

`VoucherTokenMarket`는 ERC-1155 기반의 바우처 토큰을 거래할 수 있는 스마트 컨트랙트입니다. 이 컨트랙트는 다음과 같은 주요 기능을 제공합니다.

1. 바우처 토큰 판매
2. 바우처 토큰 판매 취소
3. USDT 토큰을 통한 바우처 토큰 구매

---

### 함수 목록 및 설명

#### **1. 생성자**

**설명:** 컨트랙트를 초기화하며 토큰의 기본 정보와 수수료 관리 컨트랙트 주소를 설정합니다.

**파라미터**

- `_name` (string) : 토큰 이름

- `_symbol` (string) : 토큰 이름

- `_feeManager` (address) : 수수료 관리자 컨트랙트 주소

**로직**

1. 입력된 `_name`, `_symbol`, `_feeManager`를 설정합니다.
2. `_tokenIdTracker`를 1로 초기화합니다.

```solidity
constructor(
    string memory _name,
    string memory _symbol,
    address _feeManager
) ERC1155("") {
    name = _name;
    symbol = _symbol;
    feeManager = _feeManager;
    _tokenIdTracker.increment(); // Start token ID from 1
}
```

------

#### **2. `verifyVoucherContract`**

**설명:** 운영자 권한으로 판매가 가능하도록 Voucher Contract를 추가합니다.

**파라미터**

- **`_voucherContract` (`address`)**: 검증 등록할 Voucher Contract 주소

**로직**

1. 호출자가 오퍼레이터 권한을 가진지 확인합니다.
2. `_voucherContract` 주소를 검증된 상태로 변경합니다.

```solidity
function verifyVoucherContract(address _voucherContract) external operatorsOnly {
        voucherContractMap[_voucherContract] = true;
        emit VerificationVoucherContract(_voucherContract, true);
}
```

------

### **3. `unVerifyVoucherContract`**

**설명:** 운영자 권한으로 판매가 불가능하도록 Voucher Contract를 추가합니다.

**파라미터**

- **`_voucherContract` (`address`)**: 검증을 해제할 Voucher Contract 주소

**로직**

1. 호출자가 오퍼레이터 권한을 가진지 확인합니다.
2. `_voucherContract` 주소를 검증 해제된 상태로 변경합니다.

```solidity
function unVerifyVoucherContract(address _voucherContract) external operatorsOnly {
        voucherContractMap[_voucherContract] = false;
        emit VerificationVoucherContract(_voucherContract, false);
}
```

------

### **4. `place`**

**설명:** 바우처를 마켓에 등록합니다.

**파라미터**

- `_tokenId` (uint256): 바우처의 토큰 ID
- `_amount` (uint256): 등록할 바우처 수량
- `_voucherContract` (address): 바우처 컨트랙트 주소
- `_perTokenPrice` (uint256): 바우처 1개의 가격 (USDT 단위)

**로직**

1. `voucherContractMap`를 통해 컨트랙트가 검증되었는지 확인합니다.
2. 등록할 바우처의 수량이 최소 수량 이상인지 확인합니다.
3. 화이트리스트를 통해 등록자의 자격을 검증합니다.
4. 바우처의 가격이 최소 가격 이상인지 확인합니다.
5. 고유한 마켓 ID를 생성하여 `_marketItemMap`에 등록합니다.
6. ERC1155 바우처 토큰을 컨트랙트로 전송합니다.
7. `TokenPlaced` 이벤트를 실행합니다.

#### 함수 코드

```solidity
function place(
    uint256 _tokenId,
    uint256 _amount,
    address _voucherContract,
    uint256 _perTokenPrice
) external {
    require(voucherContractMap[_voucherContract], "Invalid voucher contract");
    require(_amount >= minVoucherAmount, "Amount below minimum limit");
    require(IWhitelist(whitelistManager).isWhitelist(_voucherContract, _tokenId, msg.sender), "Not in whitelist");
    require(_perTokenPrice >= IPrice(_voucherContract).getCarbonPrice(_tokenId), "Price below minimum");

    _marketItemIds.increment();
    uint256 marketId = _marketItemIds.current();

    _marketItemMap[marketId] = MarketItem(
        _voucherContract,
        _tokenId,
        _amount,
        _perTokenPrice,
        msg.sender
    );

    IERC1155(_voucherContract).safeTransferFrom(msg.sender, address(this), _tokenId, _amount, "");
    emit TokenPlaced(_voucherContract, marketId, _tokenId, _amount, msg.sender, _perTokenPrice);
}
```

---

### 5. `unPlace`

**설명** 마켓에 등록된 바우처를 제거하거나 수량을 줄입니다

**파라미터**

- `_marketId` (uint256): 마켓 상품의 ID.
- `_amount` (uint256): 제거할 바우처 수량.

**로직**

1. 호출자가 상품의 소유자이거나 오퍼레이터인지 확인합니다.
2. 제거하려는 수량이 상품에 등록된 수량 이내인지 확인합니다.
3. 등록된 바우처 수량에서 `_amount`를 차감합니다.
4. ERC1155 토큰을 호출자에게 반환합니다.
5. `TokenUnPlaced` 이벤트를 실행합니다.

```solidity
function unPlace(uint256 _marketId, uint256 _amount) external {
    require(_amount > 0, "Amount must be greater than zero");

    MarketItem storage marketItem = _marketItemMap[_marketId];
    require(
        marketItem.seller == msg.sender || IOperator(operatorManager).isOperator(msg.sender),
        "Not authorized"
    );
    require(marketItem.amount >= _amount, "Insufficient amount");

    marketItem.amount = marketItem.amount.sub(_amount);
    IERC1155(marketItem.voucherContract).safeTransferFrom(address(this), msg.sender, marketItem.tokenId, _amount, "");
    emit TokenUnPlaced(marketItem.voucherContract, _marketId, marketItem.tokenId, _amount, marketItem.amount, marketItem.seller, marketItem.price);
}
```

------

### 6. `purchaseInUSDT`

**설명** USDT를 사용하여 바우처를 구매합니다.

**파라미터**

- `_marketId` (uint256): 마켓 상품의 ID
- `_amount` (uint256): 구매할 바우처 수량

**로직**

1. 구매하려는 수량이 최소 수량 이상인지 확인합니다.
2. 구매자의 화이트리스트 자격을 검증합니다.
3. 상품에 등록된 바우처 수량이 충분한지 확인합니다.
4. 구매에 필요한 총 가격과 수수료를 계산합니다.
5. 구매자가 충분한 USDT를 보유하고 있는지 확인합니다.
6. 판매자 및 수수료 관리자로 USDT를 전송합니다.
7. 구매자에게 ERC1155 바우처 토큰을 전송합니다.
8. `TokenSold` 이벤트를 실행합니다.

```solidity
function purchaseInUSDT(uint256 _marketId, uint256 _amount) external {
    require(_amount >= minVoucherAmount, "Amount below minimum limit");

    MarketItem storage marketItem = _marketItemMap[_marketId];
    require(IWhitelist(whitelistManager).isWhitelist(marketItem.voucherContract, marketItem.tokenId, msg.sender), "Not in whitelist");
    require(marketItem.amount >= _amount, "Insufficient voucher amount");

    uint256 totalPrice = marketItem.price.mul(_amount);
    uint256 feeAmount = IFeeManager(feeManager).feeAmount(totalPrice);
    uint256 remainAmount = totalPrice.sub(feeAmount);

    require(IERC1155(marketItem.voucherContract).balanceOf(address(this), marketItem.tokenId) >= _amount, "Contract has insufficient tokens");
    require(IERC20(usdtContractAddress).balanceOf(msg.sender) >= totalPrice, "Insufficient USDT balance");

    marketItem.amount = marketItem.amount.sub(_amount);
    ERC20(usdtContractAddress).safeTransferFrom(msg.sender, marketItem.seller, remainAmount);
    ERC20(usdtContractAddress).safeTransferFrom(msg.sender, IFeeManager(feeManager).feeAddress(), feeAmount);
    IERC1155(marketItem.voucherContract).safeTransferFrom(address(this), msg.sender, marketItem.tokenId, _amount, "");

    emit TokenSold(marketItem.voucherContract, _marketId, marketItem.tokenId, _amount, msg.sender, marketItem.price);
}
```

### 핵심 스마트 컨트랙트 관계도

![Voucher Token Dependency](https://github.com/user-attachments/assets/8dc8e41d-0d3c-4700-b7a8-ce3d8003e721)
