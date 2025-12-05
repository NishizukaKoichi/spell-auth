# spell-auth

**Identity & Session Management**

## 責務

- Passkey (WebAuthn) 登録・ログイン
- `user_id` と `credential_id` の台帳管理
- セッション用 JWT (`user_token`) 発行・更新
- JWKS 提供

## やらないこと

- Intent 署名（→ `spell-intent`）
- 実行可否の判断（→ `spell-core`）
- 課金（→ `spell-billing`）

## Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ POST /register
       │ POST /login
       │ POST /refresh
       ▼
┌─────────────────┐
│   spell-auth    │
│                 │
│ - WebAuthn RP   │
│ - User Store    │
│ - JWT Issuer    │
└─────────────────┘
```

## API Endpoints

- `POST /register` - Passkey registration
- `POST /login` - Passkey authentication
- `POST /refresh` - JWT refresh
- `GET /.well-known/jwks.json` - Public keys for JWT verification

## Tech Stack

- Node.js / TypeScript
- SimpleWebAuthn
- jose (JWT)
- PostgreSQL (user/credential store)
