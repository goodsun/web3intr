# スマートコントラクト設計

## 1. コントラクト構成

### 1.1 MembershipSBT.sol
会員証SBT（Soul Bound Token）の発行・管理コントラクト

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/metatx/ERC2771Context.sol";

contract MembershipSBT is ERC721, Ownable, ReentrancyGuard, ERC2771Context {
    uint256 private _tokenIdCounter;
    uint256 public constant INITIAL_FUND_AMOUNT = 0.03 ether;
    
    mapping(address => bool) public hasMembership;
    mapping(uint256 => uint256) public mintTimestamp;
    
    event MembershipMinted(address indexed to, uint256 tokenId);
    event InitialFundSent(address indexed to, uint256 amount);
    
    constructor(address trustedForwarder) 
        ERC721("Web3 Introducer Membership", "W3IM")
        ERC2771Context(trustedForwarder)
        Ownable(msg.sender) 
    {}
    
    // SBT特性：転送不可
    function _update(
        address to,
        uint256 tokenId,
        address auth
    ) internal override returns (address) {
        address from = _ownerOf(tokenId);
        
        // ミント時のみ許可（from == address(0)）
        require(from == address(0), "SBT: Transfer not allowed");
        
        return super._update(to, tokenId, auth);
    }
    
    // メンバーシップ発行（メタトランザクション対応）
    function mintMembership() external nonReentrant {
        address recipient = _msgSender();
        require(!hasMembership[recipient], "Already has membership");
        require(address(this).balance >= INITIAL_FUND_AMOUNT, "Insufficient contract balance");
        
        uint256 tokenId = _tokenIdCounter++;
        hasMembership[recipient] = true;
        mintTimestamp[tokenId] = block.timestamp;
        
        _safeMint(recipient, tokenId);
        
        // 初期資金送付
        (bool success, ) = payable(recipient).call{value: INITIAL_FUND_AMOUNT}("");
        require(success, "Initial fund transfer failed");
        
        emit MembershipMinted(recipient, tokenId);
        emit InitialFundSent(recipient, INITIAL_FUND_AMOUNT);
    }
    
    // コントラクトへの資金補充
    function fundContract() external payable onlyOwner {
        require(msg.value > 0, "Must send some ETH");
    }
    
    // 残高確認
    function getContractBalance() external view returns (uint256) {
        return address(this).balance;
    }
    
    // メタトランザクション用のmsgSenderオーバーライド
    function _msgSender() internal view override(Context, ERC2771Context) returns (address) {
        return ERC2771Context._msgSender();
    }
    
    function _msgData() internal view override(Context, ERC2771Context) returns (bytes calldata) {
        return ERC2771Context._msgData();
    }
    
    // メタデータURI（IPFS想定）
    function _baseURI() internal pure override returns (string memory) {
        return "ipfs://QmXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/";
    }
}
```

### 1.2 MembershipRegistry.sol
会員情報の管理・照会用コントラクト

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract MembershipRegistry is AccessControl {
    bytes32 public constant REGISTRAR_ROLE = keccak256("REGISTRAR_ROLE");
    
    struct MemberInfo {
        uint256 tokenId;
        uint256 joinedAt;
        string metadata; // オフチェーンメタデータのハッシュ
        bool isActive;
    }
    
    mapping(address => MemberInfo) public members;
    address[] public memberList;
    
    event MemberRegistered(address indexed member, uint256 tokenId);
    event MemberUpdated(address indexed member, string metadata);
    event MemberDeactivated(address indexed member);
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(REGISTRAR_ROLE, msg.sender);
    }
    
    function registerMember(
        address member,
        uint256 tokenId,
        string memory metadata
    ) external onlyRole(REGISTRAR_ROLE) {
        require(members[member].tokenId == 0, "Member already registered");
        
        members[member] = MemberInfo({
            tokenId: tokenId,
            joinedAt: block.timestamp,
            metadata: metadata,
            isActive: true
        });
        
        memberList.push(member);
        emit MemberRegistered(member, tokenId);
    }
    
    function updateMemberMetadata(
        address member,
        string memory metadata
    ) external onlyRole(REGISTRAR_ROLE) {
        require(members[member].tokenId != 0, "Member not found");
        members[member].metadata = metadata;
        emit MemberUpdated(member, metadata);
    }
    
    function deactivateMember(address member) external onlyRole(REGISTRAR_ROLE) {
        require(members[member].tokenId != 0, "Member not found");
        members[member].isActive = false;
        emit MemberDeactivated(member);
    }
    
    function getMemberCount() external view returns (uint256) {
        return memberList.length;
    }
    
    function isMember(address account) external view returns (bool) {
        return members[account].tokenId != 0 && members[account].isActive;
    }
}
```

## 2. デプロイ設計

### 2.1 デプロイ手順

1. **Biconomy Forwarderアドレス取得**
   - Mumbai: `0x69015912AA33720b842dCD6aC059Ed623F28d9f7`
   - Mainnet: `0x86C80a8aa58e0A4fa09A69624c31Ab2a6CAD56b8`

2. **コントラクトデプロイ順序**
   ```javascript
   // 1. MembershipSBTデプロイ
   const MembershipSBT = await ethers.getContractFactory("MembershipSBT");
   const sbt = await MembershipSBT.deploy(FORWARDER_ADDRESS);
   
   // 2. MembershipRegistryデプロイ
   const MembershipRegistry = await ethers.getContractFactory("MembershipRegistry");
   const registry = await MembershipRegistry.deploy();
   
   // 3. 初期資金送付（0.3 POL = 10人分）
   await owner.sendTransaction({
     to: sbt.address,
     value: ethers.parseEther("0.3")
   });
   
   // 4. RegistryにSBTコントラクトの権限付与
   await registry.grantRole(REGISTRAR_ROLE, apiWalletAddress);
   ```

### 2.2 ガス最適化

- **バッチミント非対応**: SBTは個別発行のみ（悪用防止）
- **ストレージ最適化**: 必要最小限のデータのみオンチェーン
- **イベント活用**: 詳細データはイベントログから取得

## 3. セキュリティ考慮事項

### 3.1 一般的な脆弱性対策

1. **Reentrancy攻撃**
   - `ReentrancyGuard`使用
   - Check-Effects-Interactions パターン遵守

2. **整数オーバーフロー**
   - Solidity 0.8.x の自動チェック機能

3. **アクセス制御**
   - OpenZeppelin `Ownable`/`AccessControl`
   - 役割ベースの権限管理

### 3.2 SBT固有の考慮事項

1. **転送不可の実装**
   - `_update`関数でfromアドレスチェック
   - `approve`/`transferFrom`も実質無効化

2. **二重発行防止**
   - `hasMembership`マッピングでチェック
   - 1アドレス1トークンのみ

3. **メタトランザクション署名検証**
   - Biconomy Forwarderの信頼性に依存
   - `_msgSender()`で実際の送信者取得

## 4. アップグレード戦略

### 4.1 プロキシパターン（将来対応）

現時点では不変コントラクトとしてデプロイするが、将来的にはOpenZeppelin Upgradesを使用：

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MembershipSBTV2 is 
    ERC721Upgradeable, 
    OwnableUpgradeable, 
    UUPSUpgradeable 
{
    function _authorizeUpgrade(address) internal override onlyOwner {}
}
```

### 4.2 マイグレーション計画

1. 新コントラクトデプロイ
2. 既存SBT保有者のスナップショット
3. 新コントラクトでの再発行（ガス代運営負担）
4. 旧コントラクトの機能停止

## 5. 監査チェックリスト

- [ ] Slither静的解析パス
- [ ] MythX脆弱性スキャン
- [ ] 単体テスト100%カバレッジ
- [ ] ガス消費量ベンチマーク
- [ ] メインネットフォークテスト
- [ ] 外部監査（予算次第）

## 6. 運用考慮事項

### 6.1 初期資金管理

- **補充タイミング**: 残高が0.3 POL以下で通知
- **自動補充**: Chainlinkキーパーで実装可能
- **資金源**: 運営ウォレットから定期補充

### 6.2 モニタリング

- **The Graph**: サブグラフでイベント監視
- **Defender**: 異常検知アラート
- **Dune Analytics**: 利用状況ダッシュボード

### 6.3 緊急時対応

- **Pause機能**: 実装しない（SBTの性質上不要）
- **資金引き出し**: Owner権限で残余資金回収可能
- **コントラクト無効化**: 新バージョンへの移行で対応