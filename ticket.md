# Ticket: spell-auth

## チケット情報

- **リポジトリ**: spell-auth
- **技術スタック**: TypeScript / Node.js / Express / SimpleWebAuthn / jose / PostgreSQL
- **仕様書パス**: ../README.md, ./README.md
- **ビルドコマンド**: `pnpm build`
- **テストコマンド**: `pnpm test`
- **その他コマンド**: `pnpm lint`

---

## 実装対象チケット

（ここにチケット本文を貼り付けてください）

例:
```
[SPELL-AUTH-001] Passkey 登録・ログイン API の実装

## 概要
WebAuthn を使った Passkey 登録・ログイン機能を実装する。

## 受け入れ条件
- [ ] POST /register - ユーザー登録 + Passkey 登録
- [ ] POST /login - Passkey 認証
- [ ] POST /refresh - JWT リフレッシュ
- [ ] GET /.well-known/jwks.json - 公開鍵エンドポイント
- [ ] PostgreSQL にユーザー情報・クレデンシャル情報を保存
- [ ] JWT に user_id を含める
- [ ] すべてのエンドポイントがエラーハンドリングされている

## 技術仕様
- SimpleWebAuthn を使用
- JWT は RS256 で署名
- リフレッシュトークンは httpOnly cookie
```

---

## 補足情報

- 不明点は合理的な前提を置いて実装してください
- 既存コードスタイルに従ってください
- 破壊的変更が必要な場合は理由を明記してください
