# フロントエンド設計

## 1. 技術スタック詳細

### 1.1 コア技術
- **Next.js 14.2**: App Router、Server Components
- **React 18.3**: Concurrent Features
- **TypeScript 5.4**: 厳格な型チェック
- **Tailwind CSS 3.4**: ユーティリティファーストCSS
- **Shadcn/ui**: UIコンポーネントライブラリ

### 1.2 Web3関連
- **Web3Auth v8**: ソーシャルログイン統合
- **ethers.js v6**: ブロックチェーン操作
- **Biconomy SDK v3**: メタトランザクション
- **wagmi v2**: React Hooks for Ethereum
- **viem**: 型安全なEthereumクライアント

### 1.3 状態管理・データ取得
- **Zustand**: グローバル状態管理
- **TanStack Query v5**: サーバー状態管理
- **axios**: HTTP通信

## 2. ディレクトリ構造

```
src/
├── app/                      # Next.js App Router
│   ├── (auth)/              # 認証関連ページ
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/         # ダッシュボード
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── api/                 # API Routes
│   │   ├── auth/
│   │   ├── membership/
│   │   └── wallet/
│   ├── layout.tsx           # ルートレイアウト
│   └── page.tsx             # ホームページ
├── components/              # Reactコンポーネント
│   ├── ui/                  # 基本UIコンポーネント
│   ├── auth/                # 認証関連
│   ├── wallet/              # ウォレット関連
│   └── membership/          # 会員証関連
├── hooks/                   # カスタムフック
│   ├── useWeb3Auth.ts
│   ├── useMetaTx.ts
│   └── useMembership.ts
├── lib/                     # ユーティリティ
│   ├── web3/               # Web3設定
│   ├── api/                # APIクライアント
│   └── utils/              # 汎用ユーティリティ
├── store/                   # Zustand ストア
│   ├── authStore.ts
│   └── walletStore.ts
├── types/                   # TypeScript型定義
│   ├── web3.d.ts
│   └── api.d.ts
└── config/                  # 設定ファイル
    ├── chains.ts
    └── contracts.ts
```

## 3. コンポーネント設計

### 3.1 認証フロー

```typescript
// components/auth/SocialLogin.tsx
import { Web3Auth } from '@web3auth/modal';
import { CHAIN_NAMESPACES } from '@web3auth/base';

export function SocialLogin() {
  const initWeb3Auth = async () => {
    const web3auth = new Web3Auth({
      clientId: process.env.NEXT_PUBLIC_WEB3AUTH_CLIENT_ID!,
      chainConfig: {
        chainNamespace: CHAIN_NAMESPACES.EIP155,
        chainId: "0x89", // Polygon Mainnet
        rpcTarget: process.env.NEXT_PUBLIC_RPC_URL!,
      },
      web3AuthNetwork: "sapphire_mainnet",
    });

    await web3auth.initModal();
    return web3auth;
  };

  const handleLogin = async () => {
    const web3auth = await initWeb3Auth();
    const web3authProvider = await web3auth.connect();
    
    // プロバイダーからウォレット情報取得
    const ethersProvider = new ethers.BrowserProvider(web3authProvider);
    const signer = await ethersProvider.getSigner();
    const address = await signer.getAddress();
    
    // バックエンドに登録
    await registerUser(address);
  };

  return (
    <button onClick={handleLogin} className="...">
      Googleでログイン
    </button>
  );
}
```

### 3.2 メタトランザクション実装

```typescript
// hooks/useMetaTx.ts
import { Biconomy } from "@biconomy/mexa";

export function useMetaTx() {
  const [biconomy, setBiconomy] = useState<Biconomy | null>(null);

  useEffect(() => {
    const init = async () => {
      const biconomyInstance = new Biconomy(
        window.ethereum,
        {
          apiKey: process.env.NEXT_PUBLIC_BICONOMY_API_KEY!,
          contractAddresses: [MEMBERSHIP_SBT_ADDRESS],
        }
      );

      await biconomyInstance.init();
      setBiconomy(biconomyInstance);
    };

    init();
  }, []);

  const sendMetaTx = async (
    contractAddress: string,
    functionName: string,
    args: any[]
  ) => {
    if (!biconomy) throw new Error("Biconomy not initialized");

    const provider = biconomy.ethersProvider;
    const contract = new ethers.Contract(
      contractAddress,
      MembershipSBT_ABI,
      provider.getSigner()
    );

    const tx = await contract[functionName](...args);
    return tx.wait();
  };

  return { sendMetaTx, isReady: !!biconomy };
}
```

### 3.3 会員証発行UI

```typescript
// components/membership/MintButton.tsx
export function MintButton() {
  const { sendMetaTx } = useMetaTx();
  const [status, setStatus] = useState<'idle' | 'loading' | 'success' | 'error'>('idle');

  const handleMint = async () => {
    setStatus('loading');
    
    try {
      // メタトランザクションで会員証発行
      const tx = await sendMetaTx(
        MEMBERSHIP_SBT_ADDRESS,
        'mintMembership',
        []
      );

      // トランザクション完了待機
      await tx.wait();
      
      setStatus('success');
      
      // 成功通知
      toast.success('会員証が発行されました！');
      
    } catch (error) {
      setStatus('error');
      toast.error('発行に失敗しました');
    }
  };

  return (
    <div className="space-y-4">
      <button 
        onClick={handleMint}
        disabled={status === 'loading'}
        className="..."
      >
        {status === 'loading' ? '発行中...' : '会員証を発行'}
      </button>
      
      {status === 'success' && (
        <div className="p-4 bg-green-50 rounded-lg">
          <p>✅ 会員証が発行されました</p>
          <p>✅ 0.03 POLが送付されました</p>
        </div>
      )}
    </div>
  );
}
```

### 3.4 ダッシュボード

```typescript
// app/(dashboard)/page.tsx
export default function Dashboard() {
  const { address } = useAccount();
  const { data: membership } = useMembership(address);
  const { data: balance } = useBalance(address);

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-8">ダッシュボード</h1>
      
      {/* ウォレット情報 */}
      <Card>
        <CardHeader>
          <CardTitle>ウォレット情報</CardTitle>
        </CardHeader>
        <CardContent>
          <p>アドレス: {truncateAddress(address)}</p>
          <p>残高: {formatEther(balance)} POL</p>
        </CardContent>
      </Card>

      {/* 会員証情報 */}
      <Card className="mt-6">
        <CardHeader>
          <CardTitle>会員証</CardTitle>
        </CardHeader>
        <CardContent>
          {membership ? (
            <div>
              <p>Token ID: #{membership.tokenId}</p>
              <p>発行日: {formatDate(membership.mintedAt)}</p>
              <NFTImage tokenId={membership.tokenId} />
            </div>
          ) : (
            <MintButton />
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

## 4. 状態管理設計

### 4.1 認証ストア

```typescript
// store/authStore.ts
interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  web3auth: Web3Auth | null;
  login: (provider: string) => Promise<void>;
  logout: () => Promise<void>;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  isAuthenticated: false,
  web3auth: null,

  login: async (provider) => {
    const web3auth = get().web3auth;
    if (!web3auth) throw new Error("Web3Auth not initialized");

    const web3authProvider = await web3auth.connectTo(provider);
    const user = await web3auth.getUserInfo();
    
    set({ 
      user: {
        email: user.email!,
        name: user.name!,
        profileImage: user.profileImage!,
      },
      isAuthenticated: true 
    });
  },

  logout: async () => {
    const web3auth = get().web3auth;
    if (web3auth) {
      await web3auth.logout();
    }
    set({ user: null, isAuthenticated: false });
  },
}));
```

### 4.2 ウォレットストア

```typescript
// store/walletStore.ts
interface WalletState {
  address: string | null;
  balance: bigint;
  provider: ethers.BrowserProvider | null;
  signer: ethers.Signer | null;
  connectWallet: () => Promise<void>;
  updateBalance: () => Promise<void>;
}

export const useWalletStore = create<WalletState>((set, get) => ({
  address: null,
  balance: 0n,
  provider: null,
  signer: null,

  connectWallet: async () => {
    const web3authProvider = await web3auth.connect();
    const provider = new ethers.BrowserProvider(web3authProvider);
    const signer = await provider.getSigner();
    const address = await signer.getAddress();
    
    set({ provider, signer, address });
    get().updateBalance();
  },

  updateBalance: async () => {
    const { provider, address } = get();
    if (!provider || !address) return;
    
    const balance = await provider.getBalance(address);
    set({ balance });
  },
}));
```

## 5. API Routes設計

### 5.1 認証API

```typescript
// app/api/auth/register/route.ts
export async function POST(req: Request) {
  const { address, email, googleId } = await req.json();

  // ユーザー登録
  const user = await prisma.user.create({
    data: {
      walletAddress: address,
      email,
      googleId,
    },
  });

  // JWTトークン生成
  const token = jwt.sign(
    { userId: user.id, address },
    process.env.JWT_SECRET!,
    { expiresIn: '24h' }
  );

  return NextResponse.json({ token, user });
}
```

### 5.2 会員証API

```typescript
// app/api/membership/status/route.ts
export async function GET(req: Request) {
  const address = req.headers.get('x-wallet-address');
  
  // オンチェーンデータ取得
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const contract = new ethers.Contract(
    MEMBERSHIP_SBT_ADDRESS,
    MembershipSBT_ABI,
    provider
  );

  const hasMembership = await contract.hasMembership(address);
  
  if (!hasMembership) {
    return NextResponse.json({ hasMembership: false });
  }

  // トークンID取得
  const balance = await contract.balanceOf(address);
  const tokenId = await contract.tokenOfOwnerByIndex(address, 0);
  const mintTimestamp = await contract.mintTimestamp(tokenId);

  return NextResponse.json({
    hasMembership: true,
    tokenId: tokenId.toString(),
    mintedAt: new Date(Number(mintTimestamp) * 1000),
  });
}
```

## 6. パフォーマンス最適化

### 6.1 コード分割
- 動的インポートでWeb3ライブラリを遅延読み込み
- ルートベースの自動コード分割

### 6.2 キャッシング
- React Queryでブロックチェーンデータをキャッシュ
- Next.jsのISRで静的ページ生成

### 6.3 最適化テクニック
```typescript
// Web3ライブラリの遅延読み込み
const Web3AuthModal = dynamic(
  () => import('@web3auth/modal').then(mod => mod.Web3Auth),
  { ssr: false }
);

// メモ化
const MemoizedNFTImage = memo(NFTImage);

// デバウンス
const debouncedUpdateBalance = useMemo(
  () => debounce(updateBalance, 1000),
  [updateBalance]
);
```

## 7. エラーハンドリング

### 7.1 グローバルエラーバウンダリ

```typescript
// app/error.tsx
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Sentryにエラー送信
    console.error(error);
  }, [error]);

  return (
    <div className="flex h-screen items-center justify-center">
      <div className="text-center">
        <h2 className="text-2xl font-bold">エラーが発生しました</h2>
        <p className="mt-2 text-gray-600">{error.message}</p>
        <button onClick={reset} className="mt-4 btn-primary">
          再試行
        </button>
      </div>
    </div>
  );
}
```

### 7.2 Web3エラーハンドリング

```typescript
// lib/web3/errors.ts
export function handleWeb3Error(error: any): string {
  if (error.code === 4001) {
    return 'トランザクションがキャンセルされました';
  }
  if (error.code === -32603) {
    return 'ネットワークエラーが発生しました';
  }
  if (error.message?.includes('insufficient funds')) {
    return 'ガス代が不足しています';
  }
  return 'エラーが発生しました';
}
```