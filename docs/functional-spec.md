# spell-auth 機能仕様書（Passkey/WebAuthn → Session JWT Issuer）

## 1. 目的（Purpose）

`spell-auth` は、Spellシステムにおいて **身元（user_id）の確定** と **ログイン状態の維持（セッション）** を提供する認証サービスである。

* Passkey（WebAuthn）でユーザー本人性を検証する
* **短命 Access JWT** を発行して通常操作を快適にする
* **長命 Refresh Token（HttpOnly Cookie）** で Access JWT を更新できる
* 他サービス（例：`spell-core`）は **JWKS で Access JWT をオフライン検証**する

---

## 2. 非ゴール（Non-goals）

`spell-auth` は以下を **行わない**（責務外）：

* Intent（行為同意）を生成・解釈・実行許可する最終判断（別リポの責務）
* マルチリポ実行／ジョブ実行／Runner制御
* 課金、プラン、利用制限（別リポの責務）
* OAuth/SAML 等の外部IdP連携（必要なら別仕様）

---

## 3. 前提条件（Prerequisites）

### 3.1 実行・依存

* `spell-auth` は独立サービスとしてデプロイされる
* 永続化は **PostgreSQL** を使用（challenge/refresh/credential をDB管理）
* 他サービス/クライアントとは HTTP(S) API で連携

### 3.2 WebAuthn 前提

* クライアントは WebAuthn 対応ブラウザ/OSであること
* RP ID と Origin が正しく設定されていること

  * `AUTH_RP_ID`（例：`<root-domain>` 推奨）
  * `AUTH_ALLOWED_ORIGINS`（許可Origin allowlist）

---

## 4. 制約（Constraints）

### 4.1 共通制約（全API）

* HTTPS 必須
* リクエスト/レスポンスは **JSONのみ**

  * `Content-Type: application/json` 必須（ボディありの場合）
* CORS/Origin 制御（allowlist一致のみ許可）

  * `Access-Control-Allow-Origin`: allowlist一致時のみ、要求Originを返す
  * `Access-Control-Allow-Credentials: true`
  * `Vary: Origin`
* Cookie を使うエンドポイント（refresh/logout）は **Origin allowlist一致必須**
* レートリミット必須（詳細は 10章）

### 4.2 WebAuthn 制約

* `userVerification = "required"` 固定
* `attestation = "none"` 固定
* `expectedRPID = AUTH_RP_ID`
* `expectedOrigin` は `AUTH_ALLOWED_ORIGINS` に限定

### 4.3 トークン制約

* Access JWT は **最小権限**（身元を表すのみ）
* Refresh Token は **DB管理**（生値は保存しない）
* Refresh Token は **Rotation必須**（更新時に旧トークンを失効し、新トークンを発行）

### 4.4 ステートレス制約

* サーバはインメモリに依存しない
  challenge 等は **DBへ保存**し `challenge_id` で参照する

---

## 5. 成功条件（Success Criteria：全体）

本サービスの成功は以下が満たされること：

1. Passkey登録：options→verifyで user と credential が作成される
2. Passkeyログイン：options→verifyで user を特定し access_token を返す
3. ログイン維持：refresh cookie だけで access_token を更新できる（rotation動作）
4. ログアウト：refresh token が失効し cookie が削除される
5. JWKS：`/.well-known/jwks.json` を取得でき、他サービスがJWTを検証できる
6. BAN：凍結ユーザーは login/refresh/verify を拒否し、既存refreshを一括失効できる
7. 監査ログ：主要イベントが記録される（秘匿情報は記録しない）

---

## 6. 提供機能一覧（Functional Scope）

* F1: パスキー登録（初回登録）
* F2: パスキーログイン
* F3: セッション更新（refresh）
* F4: ログアウト（refresh失効）
* F5: セッション検証（access token verify）
* F6: JWKS公開（署名検証用）
* F7: 多端末対応（passkey追加／一覧／削除）
* F8: BAN/強制ログアウト
* F9: 監査ログ
* F10: ヘルスチェック

---

## 7. API 仕様（入力／出力／前提条件／制約／成功条件）

### 7.1 共通レスポンス（推奨）

成功：APIごとの定義に従う  
失敗：以下形式を推奨（実装で統一すること）

```json
{
  "error": {
    "code": "invalid_request",
    "message": "human readable message"
  },
  "request_id": "uuid"
}
```

---

### F1: パスキー登録（初回登録）

#### (1) `POST /api/auth/register/options`

**目的**：WebAuthn CreationOptions 発行（challenge保存）

**入力（Request JSON）**

```json
{
  "email": "user@example.com",
  "display_name": "Alice"
}
```

**出力（Response JSON）**

```json
{
  "challenge_id": "uuid",
  "publicKey": { /* PublicKeyCredentialCreationOptions(JSON) */ }
}
```

**前提条件**

* HTTPS
* Origin が allowlist に含まれる

**制約**

* `attestation = "none"`
* `userVerification = "required"`
* `rpId = AUTH_RP_ID`
* `challenge_id` と challenge を DB に保存（期限つき）
* 同一ユーザーの既存 credential があれば `excludeCredentials` に列挙する

**成功条件**

* `challenge_id` が発行され、後続 verify で検証可能な状態で DB に保存される

---

#### (2) `POST /api/auth/register/verify`

**目的**：登録レスポンス検証 → user作成/credential保存 → トークン発行

**入力（Request JSON）**

```json
{
  "challenge_id": "uuid",
  "email": "user@example.com",
  "display_name": "Alice",
  "credential": { /* WebAuthn registration credential JSON */ }
}
```

**出力（Response JSON）**

```json
{
  "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" },
  "access_token": "jwt"
}
```

**追加出力（Set-Cookie）**

* Refresh Token を HttpOnly Cookie に設定

  * `HttpOnly; SameSite=Lax; Path=/; Secure(本番); Max-Age=<refresh_ttl>`

**前提条件**

* `challenge_id` が DB に存在し、期限内
* Origin/RP ID が正しい
* credential が WebAuthn として検証できる

**制約**

* `challenge_id` は使い捨て（成功/失敗問わず失効させる）
* `credential_id` はユニーク
* `counter(signCount)` を保存

**成功条件**

* user が作成される（user_id発行）
* credential が user に紐づいて保存される
* access_token が返り、refresh_cookie がセットされる

**代表的な失敗**

* 400: invalid_request
* 401: invalid_webauthn_response / origin_mismatch / rpId_mismatch
* 409: challenge_not_found / challenge_expired / credential_already_registered

---

### F2: パスキーログイン

#### (1) `POST /api/auth/login/options`

**目的**：WebAuthn RequestOptions 発行（challenge保存）

**入力**

```json
{ "user_hint": "user@example.com" }
```

**出力**

```json
{
  "challenge_id": "uuid",
  "publicKey": { /* PublicKeyCredentialRequestOptions(JSON) */ }
}
```

**前提条件**

* HTTPS
* Origin allowlist一致

**制約**

* `userVerification = "required"`
* `challenge_id` を DB 保存（期限つき）
* `user_hint` があり該当ユーザーが存在する場合：

  * `allowCredentials` に当該ユーザーの credential を列挙
* `user_hint` 無し/不明：

  * `allowCredentials` は空（discoverable credential を許容）

**成功条件**

* 後続 verify に使用できる `challenge_id` が発行され DB に保存される

---

#### (2) `POST /api/auth/login/verify`

**目的**：Assertion検証 → user特定 → トークン発行

**入力**

```json
{
  "challenge_id": "uuid",
  "user_hint": "user@example.com",
  "credential": { /* WebAuthn assertion JSON */ }
}
```

**出力**

```json
{
  "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" },
  "access_token": "jwt"
}
```

**追加出力（Set-Cookie）**

* Refresh Token cookie を設定（Rotationは refresh 時に行う）

**前提条件**

* `challenge_id` が DB に存在し期限内
* WebAuthn assertion が検証できる
* credential が DB に存在し user を特定できる

**制約**

* `challenge_id` は使い捨て
* `counter` を更新（リプレイ耐性）
* `is_banned=true` の user は拒否

**成功条件**

* user が特定される
* access_token が返る
* refresh_cookie がセットされる

**代表的な失敗**

* 401: invalid_assertion / challenge_expired / challenge_not_found
* 403: user_banned

---

### F3: セッション更新（Refresh）

#### `POST /api/auth/token/refresh`

**目的**：refresh_cookie により access_token を再発行（Rotation）

**入力**

* なし（Refresh Token は Cookie から取得）

**出力**

```json
{ "access_token": "jwt" }
```

**追加出力（Set-Cookie）**

* Rotation により refresh_cookie を更新（新トークンに差し替え）

**前提条件**

* refresh_cookie が存在
* DB に一致する `token_hash` が存在し、期限内かつ未revoked
* Origin allowlist一致（Cookie利用エンドポイント）

**制約**

* Refresh Token Rotation 必須：

  * 旧 token を revoked（`revoked_at` 記録）
  * `replaced_by_token_hash` を保存して追跡可能にする
  * 新 token を発行し cookie も更新
* BAN user は拒否

**成功条件**

* 新しい access_token が返る
* refresh_cookie が新しいものに更新され、旧トークンは無効化される

**代表的な失敗**

* 401: refresh_missing / refresh_expired / refresh_revoked
* 403: user_banned

---

### F4: ログアウト

#### `POST /api/auth/logout`

**目的**：refresh token を失効し cookie を削除する

**入力**

* なし（refresh_cookie）

**出力**

* `204 No Content`

**前提条件**

* Origin allowlist一致（Cookie利用）
* refresh_cookie が存在する場合は DB に照合可能

**制約**

* refresh_token を revoked（`revoked_at`）
* cookie は `Max-Age=0` 等で削除

**成功条件**

* cookie が消える
* DB 上の refresh token が失効状態になる（または見つからない場合も同等に扱う）

---

### F5: セッション検証（クライアント用）

#### `GET /api/auth/verify`

**目的**：access_token を検証し user を返す（デバッグ/クライアント向け）

**入力**

* Header: `Authorization: Bearer <access_token>`

**出力**

```json
{ "user": { "id": "uuid", "email": "user@example.com", "display_name": "Alice" } }
```

**前提条件**

* access_token が存在

**制約**

* JWT署名・`exp`・`iss`・`aud` を必ず検証
* BAN user は拒否

**成功条件**

* JWT から user_id を特定し、user を返す

---

### F6: JWKS 公開

#### `GET /.well-known/jwks.json`

**目的**：他サービスが access_token を検証するための公開鍵配布

**入力**

* なし

**出力**

```json
{
  "keys": [
    { "kty": "RSA", "kid": "key-id", "use": "sig", "alg": "RS256", "...": "..." }
  ]
}
```

**前提条件**

* なし（公開）

**制約**

* `kid` は必須
* keys配列で返す（ローテーション前提）

**成功条件**

* 他サービスが JWKS を使って JWT を検証できる

---

### F7: 多端末対応（Passkey 管理）

#### (1) `POST /api/auth/passkeys/add/options`

**目的**：ログイン済み user に追加パスキー登録 options を発行

**入力**

* Header: `Authorization: Bearer <access_token>`

**出力**

```json
{
  "challenge_id": "uuid",
  "publicKey": { /* CreationOptions */ }
}
```

**前提条件**

* access_token が有効
* BAN user ではない

**制約**

* challenge を DB 保存（期限つき、type=add_passkey）
* 既存 credential を exclude

**成功条件**

* 追加登録に必要な options が取得できる

#### (2) `POST /api/auth/passkeys/add/verify`

**入力**

```json
{ "challenge_id": "uuid", "credential": { /* registration credential */ } }
```

**出力**

```json
{ "ok": true }
```

**成功条件**

* user に新 credential が追加される

#### (3) `GET /api/auth/passkeys`

**入力**

* `Authorization: Bearer <access_token>`

**出力**

```json
{
  "passkeys": [
    { "id": "uuid", "credential_id": "base64url", "created_at": "iso", "last_used_at": "iso|null" }
  ]
}
```

#### (4) `POST /api/auth/passkeys/remove`

**入力**

```json
{ "passkey_id": "uuid" }
```

**出力**

```json
{ "ok": true }
```

**制約**

* 最後の1本削除は可能だが、以後ログイン不能になり得る（UIで強確認が必須）
* サービス側は削除を拒否する必要はない（ただし監査ログ必須）

---

### F8: BAN / 強制ログアウト（管理操作）

※ 管理APIの具体I/Fはこの文書では「必須機能」として規定し、認証方式は別途（運用要件）で定める。

**要件**

* user を BAN すると、以後 login/refresh/verify を 403 で拒否する
* BAN時に当該 user の refresh_tokens を一括 revoke し、強制ログアウトする

**成功条件**

* BAN 直後から refresh が通らない（強制失効）

---

### F9: 監査ログ

**入力**：各イベント発生  
**出力**：監査ログへの記録

**制約**

* JWT本文／refresh生値／秘密鍵はログ禁止
* できれば `request_id` を付与して相関可能

**成功条件**

* register/login/refresh/logout/credential操作/BAN が追跡可能

---

### F10: ヘルスチェック

#### `GET /api/health`

**出力**

```json
{ "status": "ok" }
```

**成功条件**

* 監視がサービス稼働を検知できる

---

## 8. トークン仕様（入力／出力の補助定義）

### 8.1 Access JWT（出力）

* 形式：JWS（RS256）
* Header：

  * `alg = RS256`
  * `kid = <key-id>`（必須）
* Claims（必須）：

  * `sub`: user_id（UUID）
  * `iss`: `AUTH_ISSUER`
  * `aud`: `AUTH_AUDIENCE`（例 `"spell"`）
  * `iat`, `exp`
* TTL：`AUTH_ACCESS_TOKEN_TTL_SEC`（例 900秒）

### 8.2 Refresh Token（出力：Cookie）

* 生値は **保存しない**
* DBには `token_hash` のみ保存
* TTL：`AUTH_REFRESH_TOKEN_TTL_SEC`（例 2592000秒）
* Rotation：refresh成功時に必ず更新

---

## 9. データ仕様（DB）

### 9.1 users

* `id` UUID PK
* `email` unique nullable
* `display_name` nullable
* `is_banned` boolean default false
* `created_at`, `updated_at`

### 9.2 webauthn_credentials

* `id` UUID PK
* `user_id` FK
* `credential_id` unique（base64url）
* `public_key` bytea
* `counter` int
* `transports` text[] nullable
* `created_at`
* `last_used_at` nullable

### 9.3 refresh_tokens

* `id` UUID PK
* `user_id` FK
* `token_hash` unique
* `expires_at`
* `revoked_at` nullable
* `replaced_by_token_hash` nullable
* `created_at`

### 9.4 webauthn_challenges

* `id` UUID PK（challenge_id）
* `type` enum（register/login/add_passkey）
* `challenge` text
* `user_id` nullable
* `expires_at`
* `created_at`

---

## 10. セキュリティ仕様（制約の補強）

### 10.1 レートリミット（最低限の固定値）

* options系：30回/10分/IP
* verify系：10回/10分/IP
* refresh：60回/10分/IP（逸脱時は429）

### 10.2 Cookie 属性

* HttpOnly: true
* SameSite: Lax
* Secure: 本番 true
* Path: `/`
* 名称例：`spell_refresh`（サービス内で固定）

---

## 11. 運用設定（前提条件：環境変数）

必須：

* `DATABASE_URL`
* `AUTH_RP_ID`
* `AUTH_ALLOWED_ORIGINS`（CSV）
* `AUTH_ISSUER`
* `AUTH_AUDIENCE`
* `AUTH_JWT_PRIVATE_KEY_PEM`
* `AUTH_JWT_PUBLIC_KEY_PEM`
* `AUTH_ACCESS_TOKEN_TTL_SEC`
* `AUTH_REFRESH_TOKEN_TTL_SEC`
* `AUTH_COOKIE_SECURE`

---

## 12. 受け入れ（成功条件：テスト観点）

* 登録：options→verifyで user/credential 作成 + access_token + refresh_cookie
* ログイン：options→verifyで user特定 + access_token + refresh_cookie
* refresh：cookieのみで access更新 + rotation（旧refresh無効）
* logout：cookie削除 + DB失効
* JWKS：取得可能 + 他サービスで検証可能
* BAN：login/refresh/verify拒否 + refresh一括失効
* 監査：主要イベントが揃って記録される

---

必要ならこの機能仕様書に合わせて、**エラーコード一覧（codeの厳密定義）**と、**各APIの状態遷移（challenge→verify→失効）**も同じテンポで追記して「実装が絶対ブレない版」まで固められます。
