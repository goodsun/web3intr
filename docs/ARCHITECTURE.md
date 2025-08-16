# Web3 Introducer アーキテクチャ設計

## 概要
Google認証等のソーシャルログインからWeb3ウォレットを作成し、メタトランザクションを利用してガス代なしでSBT（Soul Bound Token）会員証を発行するシステム。

## システム構成

### 1. 技術スタック

#### フロントエンド
- **Framework**: Next.js 14 (App Router)
- **UI Library**: React 18
- **Styling**: Tailwind CSS
- **Web3 Libraries**:
  - Web3Auth SDK (ソーシャルログイン → ウォレット作成)
  - ethers.js v6 (ブロックチェーン操作)
  - Biconomy SDK v3 (メタトランザクション)

#### バックエンド
- **Runtime**: Node.js 20
- **API**: Next.js API Routes
- **Database**: PostgreSQL (ユーザー情報・トランザクション履歴)
- **Cache**: Redis (セッション管理)

#### ブロックチェーン
- **Network**: Polygon (Mumbai testnet → Mainnet)
- **Smart Contracts**: Solidity 0.8.20
- **開発環境**: Hardhat
- **メタトランザクション**: EIP-2771 (Biconomy Forwarder)

### 2. アーキテクチャ図

```
┌─────────────────┐
│   ユーザー      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  フロントエンド  │────▶│   Web3Auth      │
│   (Next.js)     │     │  (Google認証)    │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│   API Routes    │────▶│   Biconomy      │
│  (バックエンド)  │     │ (メタトランザクション) │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│   PostgreSQL    │     │  Smart Contract │
│   (ユーザーDB)   │     │   (Polygon)     │
└─────────────────┘     └─────────────────┘
```

### 3. フロー詳細

#### 3.1 ユーザー登録フロー

1. **ソーシャルログイン**
   - ユーザーがGoogleアカウントでログイン
   - Web3AuthがプロバイダーキーからEOAウォレットを生成
   - プライベートキーはWeb3Authのシャード化により安全に管理

2. **ウォレット作成**
   - Web3Authが決定的にウォレットアドレスを生成
   - ユーザーのメールアドレスとウォレットアドレスを紐付け
   - PostgreSQLにユーザー情報を保存

3. **SBT発行（メタトランザクション）**
   - フロントエンドでトランザクションデータを作成
   - Biconomy Relayerがガス代を肩代わり
   - EIP-2771準拠のForwarder経由でコントラクト実行
   - 会員証SBTがユーザーウォレットにミント

4. **初期資金送付**
   - コントラクトから0.03 POL自動送金
   - ユーザーが今後自分でトランザクションを実行可能に

#### 3.2 メタトランザクション詳細

```solidity
// EIP-2771対応コントラクト
contract MembershipSBT is ERC2771Context {
    // Biconomy Forwarderを信頼
    constructor(address trustedForwarder) 
        ERC2771Context(trustedForwarder) {}
    
    // メタトランザクション経由でのミント
    function mintMembership() external {
        address user = _msgSender(); // 実際の送信者を取得
        _mint(user, tokenId);
        // 0.03 POL送金
        payable(user).transfer(0.03 ether);
    }
}
```

### 4. セキュリティ設計

#### 4.1 認証・認可
- **JWT**: Web3Authのトークンを検証
- **セッション管理**: Redis（有効期限24時間）
- **CORS設定**: 指定ドメインのみ許可

#### 4.2 ウォレットセキュリティ
- プライベートキーはクライアントに保存しない
- Web3Authのシャード化により分散管理
- ソーシャルリカバリー機能

#### 4.3 スマートコントラクトセキュリティ
- OpenZeppelin標準実装を使用
- Reentrancy Guard実装
- アクセス制御（Ownable）
- 監査済みBiconomy Forwarder使用

### 5. データベース設計

#### Users テーブル
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    wallet_address VARCHAR(42) UNIQUE NOT NULL,
    google_id VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Transactions テーブル
```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    tx_hash VARCHAR(66) UNIQUE NOT NULL,
    type VARCHAR(50) NOT NULL, -- 'mint_sbt', 'initial_fund'
    status VARCHAR(20) NOT NULL, -- 'pending', 'success', 'failed'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 6. API設計

#### エンドポイント

```typescript
// 認証
POST   /api/auth/login    // Web3Auth認証
POST   /api/auth/logout   // ログアウト
GET    /api/auth/me       // 現在のユーザー情報

// 会員証
POST   /api/membership/mint    // SBT発行（メタトランザクション）
GET    /api/membership/status  // 発行状態確認

// ウォレット
GET    /api/wallet/balance     // 残高確認
GET    /api/wallet/nfts        // 保有NFT一覧
```

### 7. エラーハンドリング

- **リトライ機構**: トランザクション失敗時3回まで自動リトライ
- **フォールバック**: Biconomy障害時は通常トランザクションへ
- **ユーザー通知**: プッシュ通知またはメール通知
- **ログ収集**: Sentry統合でエラー追跡

### 8. スケーラビリティ

- **CDN**: 静的アセットはCloudflare
- **API**: Vercel Edge Functions（自動スケール）
- **DB**: Supabase（PostgreSQL、自動バックアップ）
- **Redis**: Upstash（サーバーレス）