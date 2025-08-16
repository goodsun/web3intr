# インフラストラクチャ・デプロイ設計

## 1. インフラ構成

### 1.1 本番環境構成

```mermaid
graph TB
    subgraph "フロントエンド"
        A[Vercel] --> B[Next.js App]
        B --> C[Edge Functions]
    end
    
    subgraph "バックエンド"
        D[Supabase] --> E[PostgreSQL]
        D --> F[Auth]
        G[Upstash Redis]
    end
    
    subgraph "ブロックチェーン"
        H[Polygon Mainnet]
        I[Biconomy Relayer]
        J[IPFS/Pinata]
    end
    
    subgraph "外部サービス"
        K[Web3Auth]
        L[The Graph]
        M[Alchemy/Infura]
    end
    
    B --> D
    B --> G
    B --> K
    C --> I
    B --> H
    B --> M
    H --> L
    B --> J
```

### 1.2 開発環境構成

- **ローカル開発**: Docker Compose
- **ブロックチェーン**: Polygon Mumbai Testnet
- **CI/CD**: GitHub Actions
- **プレビュー環境**: Vercel Preview Deployments

## 2. 環境変数管理

### 2.1 環境変数一覧

```bash
# .env.local (開発環境)

# Web3Auth
NEXT_PUBLIC_WEB3AUTH_CLIENT_ID=your_web3auth_client_id

# RPC Endpoints
NEXT_PUBLIC_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/your_key
NEXT_PUBLIC_CHAIN_ID=137

# Biconomy
NEXT_PUBLIC_BICONOMY_API_KEY=your_biconomy_api_key
BICONOMY_AUTH_TOKEN=your_auth_token

# Smart Contracts
NEXT_PUBLIC_MEMBERSHIP_SBT_ADDRESS=0x...
NEXT_PUBLIC_REGISTRY_ADDRESS=0x...

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_KEY=your_service_key

# Redis (Upstash)
REDIS_URL=redis://default:password@endpoint.upstash.io:port

# IPFS (Pinata)
PINATA_API_KEY=your_pinata_api_key
PINATA_SECRET_KEY=your_pinata_secret

# Monitoring
SENTRY_DSN=https://your_sentry_dsn
NEXT_PUBLIC_POSTHOG_KEY=your_posthog_key

# Private Keys (APIウォレット用)
API_WALLET_PRIVATE_KEY=0x...
```

### 2.2 環境別設定

```typescript
// config/environments.ts
export const config = {
  development: {
    chainId: 80001, // Mumbai
    rpcUrl: 'https://rpc-mumbai.maticvigil.com',
    contracts: {
      membershipSBT: '0x...',
      registry: '0x...',
    },
  },
  staging: {
    chainId: 137, // Polygon Mainnet
    rpcUrl: process.env.NEXT_PUBLIC_RPC_URL,
    contracts: {
      membershipSBT: process.env.NEXT_PUBLIC_MEMBERSHIP_SBT_ADDRESS,
      registry: process.env.NEXT_PUBLIC_REGISTRY_ADDRESS,
    },
  },
  production: {
    // 本番設定
  },
};
```

## 3. CI/CDパイプライン

### 3.1 GitHub Actions ワークフロー

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  # 1. テスト実行
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run type check
        run: npm run type-check
      
      - name: Run tests
        run: npm test
      
      - name: Run E2E tests
        run: npm run test:e2e

  # 2. スマートコントラクトテスト
  contract-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
      
      - name: Run contract tests
        run: |
          cd contracts
          forge test -vvv
      
      - name: Check gas snapshots
        run: forge snapshot --check

  # 3. ビルド & デプロイ
  deploy:
    needs: [test, contract-test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

### 3.2 スマートコントラクトデプロイ

```yaml
# .github/workflows/deploy-contracts.yml
name: Deploy Smart Contracts

on:
  workflow_dispatch:
    inputs:
      network:
        description: 'Deploy network'
        required: true
        default: 'mumbai'
        type: choice
        options:
          - mumbai
          - polygon

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: |
          cd contracts
          npm ci
      
      - name: Compile contracts
        run: npx hardhat compile
      
      - name: Deploy contracts
        env:
          PRIVATE_KEY: ${{ secrets.DEPLOYER_PRIVATE_KEY }}
          ALCHEMY_KEY: ${{ secrets.ALCHEMY_KEY }}
        run: |
          npx hardhat run scripts/deploy.ts --network ${{ inputs.network }}
      
      - name: Verify contracts
        run: |
          npx hardhat verify --network ${{ inputs.network }} \
            $MEMBERSHIP_SBT_ADDRESS \
            $FORWARDER_ADDRESS
```

## 4. モニタリング・ログ設計

### 4.1 アプリケーションモニタリング

```typescript
// lib/monitoring/sentry.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
  beforeSend(event) {
    // 個人情報をフィルタリング
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    return event;
  },
});

// エラー境界
export function reportError(error: Error, context?: any) {
  Sentry.captureException(error, {
    extra: context,
  });
}
```

### 4.2 ブロックチェーンモニタリング

```typescript
// lib/monitoring/blockchain.ts
import { ethers } from 'ethers';

export class BlockchainMonitor {
  private provider: ethers.Provider;
  
  constructor() {
    this.provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  }

  // トランザクション監視
  async monitorTransaction(txHash: string) {
    const receipt = await this.provider.waitForTransaction(txHash);
    
    // PostHogにイベント送信
    posthog.capture('transaction_confirmed', {
      txHash,
      gasUsed: receipt.gasUsed.toString(),
      status: receipt.status,
    });
    
    return receipt;
  }

  // コントラクトイベント監視
  subscribeToEvents() {
    const contract = new ethers.Contract(
      MEMBERSHIP_SBT_ADDRESS,
      MembershipSBT_ABI,
      this.provider
    );

    contract.on('MembershipMinted', (to, tokenId, event) => {
      console.log('New membership minted:', { to, tokenId });
      
      // Webhookで通知
      fetch('/api/webhooks/membership-minted', {
        method: 'POST',
        body: JSON.stringify({ to, tokenId, txHash: event.transactionHash }),
      });
    });
  }
}
```

### 4.3 ダッシュボード設定

```typescript
// PostHogダッシュボード設定
const dashboards = {
  // ユーザー分析
  userAnalytics: {
    widgets: [
      'Daily Active Users',
      'User Retention',
      'Conversion Funnel (Login → Mint)',
    ],
  },
  
  // トランザクション分析
  transactionAnalytics: {
    widgets: [
      'Daily Mints',
      'Gas Usage',
      'Meta Transaction Success Rate',
    ],
  },
  
  // エラー監視
  errorMonitoring: {
    widgets: [
      'Error Rate',
      'Top Errors',
      'Error by Browser',
    ],
  },
};
```

## 5. セキュリティ設定

### 5.1 Vercelセキュリティヘッダー

```json
// vercel.json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline' *.vercel-insights.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' *.supabase.co *.web3auth.io wss://*.walletconnect.org"
        }
      ]
    }
  ]
}
```

### 5.2 APIレート制限

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10秒間に10リクエスト
});

export async function middleware(request: NextRequest) {
  const ip = request.ip ?? '127.0.0.1';
  const { success } = await ratelimit.limit(ip);
  
  if (!success) {
    return new Response('Rate limit exceeded', { status: 429 });
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: '/api/:path*',
};
```

## 6. バックアップ・災害復旧

### 6.1 データベースバックアップ

```bash
# Supabaseの自動バックアップ設定
# - 日次バックアップ: 7日間保持
# - 週次バックアップ: 4週間保持
# - 月次バックアップ: 12ヶ月保持

# 手動バックアップスクリプト
#!/bin/bash
# scripts/backup-db.sh

DATE=$(date +%Y%m%d_%H%M%S)
pg_dump $DATABASE_URL > backups/db_backup_$DATE.sql
aws s3 cp backups/db_backup_$DATE.sql s3://your-backup-bucket/
```

### 6.2 スマートコントラクト状態バックアップ

```typescript
// scripts/backup-contract-state.ts
async function backupContractState() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const contract = new ethers.Contract(
    MEMBERSHIP_SBT_ADDRESS,
    MembershipSBT_ABI,
    provider
  );

  // 全トークンIDを取得
  const totalSupply = await contract.totalSupply();
  const tokenData = [];

  for (let i = 0; i < totalSupply; i++) {
    const owner = await contract.ownerOf(i);
    const timestamp = await contract.mintTimestamp(i);
    tokenData.push({ tokenId: i, owner, timestamp });
  }

  // JSONとして保存
  fs.writeFileSync(
    `backups/contract_state_${Date.now()}.json`,
    JSON.stringify(tokenData, null, 2)
  );
}
```

## 7. デプロイチェックリスト

### 7.1 事前準備
- [ ] 環境変数設定完了
- [ ] Vercelプロジェクト作成
- [ ] Supabaseプロジェクト作成
- [ ] Web3Auth設定完了
- [ ] Biconomyダッシュボード設定

### 7.2 スマートコントラクト
- [ ] テストネットデプロイ・検証
- [ ] 監査完了（必要に応じて）
- [ ] メインネットデプロイ
- [ ] Etherscan検証
- [ ] 初期資金送付

### 7.3 アプリケーション
- [ ] ビルドエラーなし
- [ ] 型チェック通過
- [ ] テスト全通過
- [ ] E2Eテスト成功
- [ ] Lighthouse性能スコア確認

### 7.4 セキュリティ
- [ ] 秘密鍵の安全な管理
- [ ] CORS設定確認
- [ ] レート制限設定
- [ ] WAF設定（Cloudflare）

### 7.5 モニタリング
- [ ] Sentry設定完了
- [ ] PostHog設定完了
- [ ] アラート設定
- [ ] ダッシュボード作成