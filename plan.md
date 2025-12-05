# Plan: spell-auth

## 実装計画

このファイルは「完遂モード」で AI に渡す実装計画書です。

---

## 目的

spell-auth リポジトリの初期実装を完遂する。

---

## 実装フェーズ

### Phase 1: 環境セットアップ

- [ ] package.json に必要な依存関係を追加
  - `express`, `@simplewebauthn/server`, `jose`, `pg`, `zod` など
- [ ] TypeScript 設定 (tsconfig.json)
- [ ] ESLint / Prettier 設定
- [ ] Docker Compose (PostgreSQL)
- [ ] .env.example 作成

### Phase 2: データベーススキーマ

- [ ] users テーブル
  - id (UUID, PK)
  - username (UNIQUE)
  - created_at, updated_at
- [ ] credentials テーブル
  - id (UUID, PK)
  - user_id (FK)
  - credential_id (UNIQUE)
  - public_key
  - counter
  - transports
  - created_at

### Phase 3: 認証 API 実装

- [ ] POST /register - ユーザー登録 + Passkey 登録
- [ ] POST /login - Passkey 認証
- [ ] POST /refresh - JWT リフレッシュ
- [ ] GET /.well-known/jwks.json - 公開鍵エンドポイント

### Phase 4: JWT 発行・検証

- [ ] RS256 鍵ペア生成
- [ ] JWT 発行ロジック
- [ ] JWT 検証ミドルウェア
- [ ] リフレッシュトークン管理

### Phase 5: テスト

- [ ] ユニットテスト (各エンドポイント)
- [ ] 統合テスト (E2E)

### Phase 6: ドキュメント

- [ ] API ドキュメント
- [ ] 環境変数リスト
- [ ] デプロイ手順

---

## ファイル構成（想定）

```
spell-auth/
├── src/
│   ├── server.ts              # Entry point
│   ├── config/
│   │   ├── db.ts              # PostgreSQL client
│   │   └── keys.ts            # JWT key management
│   ├── routes/
│   │   ├── register.ts        # POST /register
│   │   ├── login.ts           # POST /login
│   │   ├── refresh.ts         # POST /refresh
│   │   └── jwks.ts            # GET /.well-known/jwks.json
│   ├── models/
│   │   ├── user.ts            # User model
│   │   └── credential.ts      # Credential model
│   ├── middleware/
│   │   └── auth.ts            # JWT verification middleware
│   └── utils/
│       └── jwt.ts             # JWT utilities
├── tests/
│   ├── register.test.ts
│   ├── login.test.ts
│   └── refresh.test.ts
├── migrations/
│   └── 001_initial.sql
├── docker-compose.yml
├── .env.example
├── tsconfig.json
├── package.json
└── README.md
```

---

## 技術的決定事項

- **WebAuthn ライブラリ**: @simplewebauthn/server
- **JWT ライブラリ**: jose
- **DB クライアント**: pg (node-postgres)
- **バリデーション**: zod
- **テストフレームワーク**: vitest
- **HTTP フレームワーク**: express

---

## 注意事項

- JWT は RS256 で署名
- リフレッシュトークンは httpOnly cookie に保存
- credential_id は base64url エンコード
- すべてのエラーは適切にハンドリング
- 環境変数で RP_ID, RP_NAME, ORIGIN などを設定可能にする

---

## 完遂モード実行時の指示

ticket.md のチケット内容と、この plan.md を参照して、上記フェーズを全て実装してください。
不明点は合理的な前提を置いて実装を完了させてください。
