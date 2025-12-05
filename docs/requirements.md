# spell-auth 要件定義書（Passkey/WebAuthn → Session JWT Issuer）

## 0. 文書情報

* 対象リポジトリ: `spell-auth`
* 目的: Passkey(WebAuthn)でユーザー認証し、セッション維持のためのJWTを発行する
* ステータス: 確定（この文書が `spell-auth` の仕様の唯一の正本）

---

## 1. 背景 / 目的

`spell-auth` は **Spell システム全体における「身元の確定」と「ログイン状態の維持」**を一手に担う認証サービスである。

* パスキー（WebAuthn）でユーザー本人性を確認する
* **短命 Access JWT** を発行し、通常操作での再サインイン頻度を下げる
* **長命 Refresh Token（HttpOnly Cookie）** により、Access JWT の更新を可能にする
* Spell 側（`spell-core` 等）は **JWKS によるオフライン検証**で Access JWT を検証する

> 重要:
> `spell-auth` は **「誰か（user_id）の確定」と「セッションの維持」**だけを提供する。
> **「この行為（Intent）を実行してよいか」という最終同意・認可・ポリシー判断は `spell-auth` の責務ではない。**（Intent は別リポの責務）

---

## 2. スコープ（やること / やらないこと）

### 2.1 やること（In Scope）

* WebAuthn（パスキー）の登録（初回）
* WebAuthn（パスキー）のログイン
* Access JWT の発行
* Refresh Token の発行・更新・失効（サーバ側管理）
* JWKS 公開（署名検証用）
* アカウント凍結（BAN）と強制ログアウト
* 監査ログ（ログイン成功/失敗、トークン更新、ログアウト等）
* レートリミット、CORS/Origin 制御などの最低限の防御

### 2.2 やらないこと（Out of Scope）

* Intent の生成、解釈、実行許可判断（最終同意）
* マルチリポ実行、ジョブ実行、Runner 管理
* 課金、プラン、利用制限（それらは別リポの責務）
* OAuth/SAML など外部IdP連携（必要になったら別途定義）

---

## 3. 用語

* **Passkey / WebAuthn**: ブラウザ＋OS＋認証器で行う公開鍵認証
* **Credential**: パスキーの識別子（credential_id）と公開鍵等の集合
* **Access JWT**: 署名付き短命トークン（通常API呼び出しで利用）
* **Refresh Token**: 長命・サーバ管理・Cookieで保持する更新用トークン
* **JWKS**: JWT 検証用公開鍵を配布するJSON（`/.well-known/jwks.json`）

---

## 4. システム境界 / 依存

### 4.1 システム境界

* `spell-auth` は独立したサービスとしてデプロイされる
* 依存先は **PostgreSQL**（永続化） のみ
* 他サービス（例: `spell-core`, `spell-client`）とはHTTPで連携する

### 4.2 想定デプロイ

* 例: `https://auth.<root-domain>`（Vercel 等）
* WebAuthn RP ID は **ルートドメイン（<root-domain>）** を推奨
  （例: `AUTH_RP_ID=<root-domain>`、Origin は `https://app.<root-domain>` 等を許可）

---

## 5. アーキテクチャ要求（重要な設計原則）

1. **ステートレス実行**: サーバはインメモリに依存しない（challenge等はDBに保存）
2. **オフライン検証可能**: JWT 検証はクライアント/他サービス側で完結（JWKS配布）
3. **最小権限**: Access JWT は「ログイン済みの身元」だけを伝える。権限を詰め込まない
4. **再サインイン頻度を下げる**: Refresh Token による更新でUXを担保
5. **複数デバイス対応**: 1 user_id に紐づく credential を複数持てる

---

## 6. 機能要件（Functional Requirements）

### FR-1 ユーザー登録（パスキー登録）

* ユーザーはパスキーを作成し、アカウントを作成できる
* 登録フローは WebAuthn の **options → verify** の2段階
* 登録完了時に `user_id` を確定し、credential を永続化する

### FR-2 ログイン（既存パスキー）

* ユーザーは既存パスキーでログインできる
* ログイン完了時に Access JWT を返し、Refresh Token をCookieに設定する

### FR-3 セッション（トークン発行・更新・失効）

* Access JWT を発行する（短命）
* Refresh Token を発行する（長命、HttpOnly Cookie）
* Refresh により Access JWT を更新できる
* Logout により Refresh Token を失効し、Cookieを削除する
* BAN（凍結）されたユーザーは login/refresh を拒否する

### FR-4 パスキー（credential）の追加・削除

* ユーザーはログイン後に追加のパスキーを登録できる（多端末対応）
* ユーザーは自身のパスキーを一覧できる
* ユーザーは不要なパスキーを削除できる
  ※最後の1つを削除する場合は「以後ログイン不可」になるため、UI/確認フローを必須とする

### FR-5 JWT 検証情報の提供（JWKS）

* `/.well-known/jwks.json` を公開し、サービス外部からJWT検証が可能であること
* 鍵ローテーションのため、`kid` を必須とする

### FR-6 監査ログ

* 以下のイベントを必ず記録する（PIIは最小化）

  * register start/finish（成功/失敗）
  * login start/finish（成功/失敗）
  * refresh（成功/失敗）
  * logout（成功/失敗）
  * credential add/remove
  * BAN/UNBAN（管理操作）
* JWT本文やRefresh Token生値はログに出さない

### FR-7 ヘルスチェック

* `GET /api/health` で稼働確認ができる

---

## 7. 外部インターフェース（API 仕様）

### 7.0 共通仕様（全エンドポイント）

* 通信: HTTPS 必須
* Content-Type: `application/json` 必須（JSONのみ）
* CORS: `Origin` を allowlist で制御し、許可された Origin のみ応答を許す

  * `Access-Control-Allow-Origin`: リクエスト Origin を反映（allowlist一致時のみ）
  * `Access-Control-Allow-Credentials: true`（Cookie利用のため）
  * `Vary: Origin` を設定
* CSRF/悪用対策:

  * Cookieを使うエンドポイント（refresh/logout）は **Origin allowlist一致**必須
  * SameSite=Lax + Origin 検証で CSRF を成立しにくくする
* レートリミットを必須（詳細はセキュリティ章）

---

### 7.1 登録（パスキー作成）

#### POST `/api/auth/register/options`

**目的**: WebAuthn CreationOptions の発行（challenge保持）

Request:

```json
{
  "email": "user@example.com",
  "display_name": "Alice"
}
```

Response:

```json
{
  "challenge_id": "uuid",
  "publicKey": { "/* PublicKeyCredentialCreationOptions(JSON) */": true }
}
```

要件:

* `challenge_id` を DB に保存（期限つき）
* `attestation` は `"none"` 固定
* `userVerification` は `"required"` 固定
* `rpId` は `AUTH_RP_ID`
* `excludeCredentials` に、同一ユーザーの既存 credential を入れる（重複登録防止）

#### POST `/api/auth/register/verify`

**目的**: 登録レスポンス検証 → ユーザー作成/credential保存 → トークン発行

Request:

```json
{
  "challenge_id": "uuid",
  "email": "user@example.com",
  "display_name": "Alice",
  "credential": { "/* WebAuthn credential JSON */": true }
}
```

Response:

```json
{
  "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" },
  "access_token": "jwt"
}
```

追加動作:

* Refresh Token を **HttpOnly Cookie** に `Set-Cookie`
* `challenge_id` は使い捨て（成功/失敗どちらでも失効させる）

エラー例（代表）:

* 400: invalid_request
* 401: invalid_webauthn_response / origin_mismatch / rpId_mismatch
* 409: challenge_expired / challenge_not_found

---

### 7.2 ログイン（パスキー認証）

#### POST `/api/auth/login/options`

Request:

```json
{ "user_hint": "user@example.com" }
```

Response:

```json
{
  "challenge_id": "uuid",
  "publicKey": { "/* PublicKeyCredentialRequestOptions(JSON) */": true }
}
```

要件:

* `user_hint` が指定され、該当ユーザーが存在する場合:

  * `allowCredentials` にそのユーザーの credential を列挙
* `user_hint` が無い/不明の場合:

  * `allowCredentials` は空（discoverable credential を許容）
* `userVerification` は `"required"` 固定
* `challenge_id` を DB に保存（期限つき）

#### POST `/api/auth/login/verify`

Request:

```json
{
  "challenge_id": "uuid",
  "user_hint": "user@example.com",
  "credential": { "/* WebAuthn assertion JSON */": true }
}
```

Response:

```json
{
  "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" },
  "access_token": "jwt"
}
```

追加動作:

* Refresh Token を HttpOnly Cookie に `Set-Cookie`
* signCount/counter を更新（リプレイ耐性）

BAN:

* `is_banned=true` の user は 403 で拒否

---

### 7.3 トークン更新

#### POST `/api/auth/token/refresh`

Request: なし（Refresh Token Cookie）

Response:

```json
{ "access_token": "jwt" }
```

要件:

* Refresh Token は **DB管理**（失効可能）
* **Refresh Token Rotation を必須**:

  * refresh 成功時、旧tokenを revoked にして新tokenを発行、Cookieを更新
* BAN user は 403、refresh token 失効/期限切れは 401

---

### 7.4 ログアウト

#### POST `/api/auth/logout`

Request: なし（Refresh Token Cookie）

Response: `204 No Content`

要件:

* 該当 Refresh Token を失効（revoked_at 記録）
* Cookie を削除（Max-Age=0）

---

### 7.5 セッション検証（クライアント用）

#### GET `/api/auth/verify`

Headers:

* `Authorization: Bearer <access_token>`

Response:

```json
{ "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" } }
```

要件:

* 署名・exp・iss・aud を必ず検証
* BAN user は 403

---

### 7.6 JWKS

#### GET `/.well-known/jwks.json`

Response:

```json
{ "keys": [ { "kty": "RSA", "kid": "key-id", "use": "sig", "alg": "RS256", "...": "..." } ] }
```

要件:

* `kid` を必須
* 将来ローテを想定し keys 配列で返す

（実装上は `GET /api/jwks` を作り、rewriteで `/.well-known/jwks.json` に合わせてもよい）

---

### 7.7 パスキー管理（多端末対応）

#### POST `/api/auth/passkeys/add/options`

Headers:

* `Authorization: Bearer <access_token>`

Response:

```json
{
  "challenge_id": "uuid",
  "publicKey": { "/* CreationOptions */": true }
}
```

#### POST `/api/auth/passkeys/add/verify`

Headers:

* `Authorization: Bearer <access_token>`

Request:

```json
{ "challenge_id": "uuid", "credential": { "/* registration credential */": true } }
```

Response:

```json
{ "ok": true }
```

#### GET `/api/auth/passkeys`

Headers:

* `Authorization: Bearer <access_token>`

Response:

```json
{
  "passkeys": [
    { "id": "uuid", "credential_id": "base64url", "created_at": "iso", "last_used_at": "iso|null" }
  ]
}
```

#### POST `/api/auth/passkeys/remove`

Headers:

* `Authorization: Bearer <access_token>`

Request:

```json
{ "passkey_id": "uuid" }
```

Response:

```json
{ "ok": true }
```

要件:

* **最後の1本削除**は明示的に許可（ただし UI 側で強い確認を必須）
  サービスは「削除した」事実を返す（復旧は別要件）

---

### 7.8 ヘルスチェック

#### GET `/api/health`

Response:

```json
{ "status": "ok" }
```

---

## 8. JWT 仕様（Access JWT）

### 8.1 署名/鍵

* アルゴリズム: **RS256**
* `kid` を header に必須
* 秘密鍵: `AUTH_JWT_PRIVATE_KEY_PEM`
* 公開鍵: `AUTH_JWT_PUBLIC_KEY_PEM`
* JWKS で公開鍵配布（`/.well-known/jwks.json`）

### 8.2 Claims

必須:

* `sub`: user_id（UUID）
* `iss`: `AUTH_ISSUER`
* `aud`: `AUTH_AUDIENCE`（例 `"spell"`）
* `iat`: 発行時刻
* `exp`: 失効時刻

推奨:

* `jti`: 一意ID（監査相関用）
* `sid`: セッションID（refresh token レコードID等）

### 8.3 有効期限

* Access JWT: **15分**（環境変数で変更可）
* Refresh Token: **30日**（環境変数で変更可）

---

## 9. データモデル（DB 要件）

### 9.1 `users`

* `id` UUID PK
* `email` text unique nullable
* `display_name` text nullable
* `is_banned` boolean default false
* `created_at`, `updated_at`

Index:

* `email` unique

### 9.2 `webauthn_credentials`

* `id` UUID PK
* `user_id` UUID FK(users.id)
* `credential_id` text unique（base64url）
* `public_key` bytea（@simplewebauthn/server が返す credentialPublicKey を保存）
* `counter` int
* `transports` text[] nullable
* `created_at`
* `last_used_at` nullable（運用上便利）

Index:

* `credential_id` unique
* `user_id` index

### 9.3 `refresh_tokens`

* `id` UUID PK
* `user_id` UUID FK(users.id)
* `token_hash` text unique（生値は保存しない）
* `expires_at` timestamp
* `revoked_at` timestamp nullable
* `created_at`
* `replaced_by_token_hash` text nullable（rotation追跡用）

Index:

* `token_hash` unique
* `user_id` index

### 9.4 `webauthn_challenges`

* `id` UUID PK（challenge_id）
* `type` enum: `register | login | add_passkey`
* `challenge` text（base64url）
* `user_id` UUID nullable（登録追加時は必須、初回登録は任意）
* `expires_at` timestamp
* `created_at`

要件:

* 期限切れ challenge は定期的に削除（バッチ or lazy cleanup）

---

## 10. セキュリティ要件（NFR）

### 10.1 WebAuthn

* `userVerification = required`
* `expectedRPID = AUTH_RP_ID`
* `expectedOrigin` は allowlist（例: `AUTH_ALLOWED_ORIGINS`）
* attestation は `none`（デフォルト）

### 10.2 Cookie（Refresh）

* HttpOnly: true
* Secure: 本番 true
* SameSite: Lax
* Path: `/`
* Max-Age/Expires: Refresh TTL に一致
* Cookie名は固定（例: `spell_refresh`）

### 10.3 レートリミット（必須）

* `/auth/*` に IP ベース＋（可能なら）user_hint/email ベース
* 例（固定値でよい）:

  * options 系: 30回/10分/IP
  * verify 系: 10回/10分/IP
  * refresh: 60回/10分/IP（過剰はブロック）
* ロックアウトは「短時間の遅延・拒否」まで（恒久BANは管理操作のみ）

### 10.4 BAN/強制ログアウト

* `is_banned=true` は login/refresh/verify を拒否
* BAN 操作時に対象ユーザーの refresh_tokens を一括 revoke（強制ログアウト）

### 10.5 ログ/秘匿

* JWT本文・Refresh生値・秘密鍵はログ出力禁止
* エラーは `request_id` を付与し追跡可能にする

---

## 11. 運用要件（Deployment/Config）

### 11.1 必須環境変数

* `DATABASE_URL`
* `AUTH_RP_ID`
* `AUTH_ALLOWED_ORIGINS`（CSV）
* `AUTH_ISSUER`
* `AUTH_AUDIENCE`
* `AUTH_JWT_PRIVATE_KEY_PEM`
* `AUTH_JWT_PUBLIC_KEY_PEM`
* `AUTH_ACCESS_TOKEN_TTL_SEC`（例 900）
* `AUTH_REFRESH_TOKEN_TTL_SEC`（例 2592000）
* `AUTH_COOKIE_SECURE`（prod=true）

### 11.2 可用性/性能

* 99.9% 稼働を目標（実装・監視は別計画）
* options/verify のp95応答は < 500ms を目標（DB込み）

---

## 12. 受け入れ基準（Acceptance Criteria）

1. 登録:

* options→verify で新 user が作成され credential が保存される
* access_token が返り、refresh_cookie がセットされる

2. ログイン:

* options→verify で既存 user が特定される
* access_token が返り、refresh_cookie がセットされる

3. refresh:

* refresh_cookie だけで access_token を更新できる
* rotation により refresh_cookie が更新され、旧トークンは無効になる

4. logout:

* refresh_cookie が消え、DB上のtokenが失効済みになる

5. JWKS:

* `/.well-known/jwks.json` が取得でき、`spell-core` がJWT検証できる

6. BAN:

* BAN ユーザーは login/refresh/verify が拒否される
* 既存 refresh_token は revoke される

---

この仕様で `spell-auth` は **「ログインを楽にする」ための JWT 基盤として完成**し、他リポ（特に `spell-intent` と `spell-core`）と責務衝突しません。

次に必要なら、同じ粒度で **`spell-intent` の要件定義書**（Intent JSON → Passkey署名 → 検証 → user_id確定／リプレイ防止まで）も同様に“確定版”として書けます。
