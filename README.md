## **VoucherToken 스마트 컨트랙트**

VoucherToken 스마트 컨트랙트는 탄소 상쇄권과 연동된 ERC-1155 기반 NFT 토큰을 발행, 관리 및 거래하기 위해 설계되었습니다. 이 컨트랙트는 다음과 같은 주요 기능을 제공합니다.

1. **운영자 기반 민팅**
2. **서명 기반 민팅 및 전송**
3. **수수료 처리**

------

### **함수 목록 및 설명**

#### **1. 생성자**

**설명:** 컨트랙트를 초기화하며 토큰 이름, 심볼, 그리고 수수료 관리 컨트랙트 주소를 설정합니다.

 **파라미터:**

- `_name` (string): 토큰 이름
- `_symbol` (string): 토큰 심볼
- `_feeManager` (address): 수수료 관리자 컨트랙트 주소

**로직:**

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

**설명**

`recoverSigner` 함수는 서명을 복구하여 서명자가 누구인지 확인하는 데 사용됩니다.
Ethereum에서 서명은 메시지의 무결성을 확인하고 특정 개인(private key)만이 해당 서명을 생성했음을 증명합니다.
이 함수는 ECDSA(Elliptic Curve Digital Signature Algorithm)를 사용하여 서명에서 서명자의 주소를 복구합니다.

**파라미터**

- `hash`  (bytes32): 서명된 데이터의 해시.
- `signature` (bytes): 서명 데이터.

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

### 핵심 스마트 컨트랙트 관계도

![Voucher Token Dependency](https://github.com/user-attachments/assets/8dc8e41d-0d3c-4700-b7a8-ce3d8003e721)
