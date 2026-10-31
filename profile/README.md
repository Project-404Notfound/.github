# ソシャゲ開発 概要書

Unity(C#)クライアントとGoバックエンドで、個人運営できる規模のソーシャルゲームを作る。最初の実装対象はログイン処理で、ゲストアカウント作成とトークン認証から始める。

## 概要

個人開発のソーシャルゲームで、クライアントは Unity (C#)、バックエンドは Go で実装する。サーバーは自前で構築するか、無料枠のあるホスティングを借りる。ゲームのジャンルやシステムは未定のため、まず「どのゲームにも必要な土台」から作る。

**最初のマイルストーン: ログイン処理**

1. アプリ初回起動でゲストアカウントを自動作成する
2. 2回目以降は保存した認証情報で自動ログインする
3. ログイン後に発行されたアクセストークンで、認証が必要なAPIを呼べる

**今回のスコープ外**(ログインの次以降): 機種変更時の引き継ぎ、Apple / Google アカウント連携、課金、ガチャ、フレンド機能。

実装は手動で行うため、このドキュメントでは設計とコードの提示までを行う。

---

## 各情報

指定済みの項目と、こちらで提案した項目(要確認)、まだ決まっていない項目を分けて整理した。

| 項目 | 内容 | 状態 |
| --- | --- | --- |
| クライアント | Unity (C#)。バージョンは Unity 6 (6000.x) を想定 | 指定済み |
| バックエンド | Go (1.22 以降) | 指定済み |
| サーバー | 個人サーバーの構築、または無料サーバーを借りる | 指定済み(具体的な選定は未了) |
| データベース | PostgreSQL | 提案 |
| 通信方式 | HTTPS + JSON の REST API | 提案 |
| 認証方式 | ゲストアカウント + シークレット照合 → JWT アクセストークン | 提案 |
| 開発環境 | Docker Compose でローカルに Go + PostgreSQL を起動 | 提案 |
| 対象プラットフォーム | iOS / Android / PC など | 未定 |
| ゲームジャンル・システム | 未定 | 未定 |
| 想定ユーザー数 | 未定(無料サーバーの選定に影響する) | 未定 |
| 開発体制 | 個人開発を想定 | 要確認 |

---

## 技術選定について

方針は「依存を少なくして、1人で保守できる構成にする」。指定された Unity と Go は活かし、周辺は標準機能と実績の多い選択肢を採った。

### 採用する技術

| 領域 | 採用 | 理由 | 代替案 |
| --- | --- | --- | --- |
| クライアント | Unity 6 (C#) | 指定。iOS / Android / PC へ同じコードで出せる | — |
| クライアント通信 | UnityWebRequest + JsonUtility | 標準機能のみで動き、追加パッケージが不要 | UniTask, Newtonsoft.Json |
| サーバー言語 | Go 1.22 以降、標準の net/http | メソッド付きルーティングが標準で使え、フレームワーク不要。単一バイナリで配置でき、ARM 向けビルドも容易 | chi, gin, echo |
| データベース | PostgreSQL (pgx) | 所持品や課金のように整合性が要る処理で、トランザクションが使える | MySQL, 最小構成なら SQLite |
| 認証 | JWT (HS256) をアクセストークンに使う | サーバー側にセッションを持たずに済む。ライブラリは golang-jwt v5 | DB保存の不透明トークン |
| ローカル開発 | Docker Compose (Go + PostgreSQL) | 本番に近い環境を1コマンドで起動できる | — |


## ログイン処理の設計

アカウント作成時にサーバーがランダムな「ログイン用シークレット」を発行し、端末が保存する。以降の起動では、ユーザーIDとシークレットを送って新しいアクセストークンを受け取る。シークレットはサーバー側ではハッシュ化して保存する。

### フロー

1. 起動時に端末へ保存済みのユーザーIDとシークレットを探す
2. **なければ(初回)**: `POST /v1/auth/guest` を呼ぶ。サーバーがユーザーを作成し、ユーザーID・シークレット・アクセストークンを返す。クライアントはIDとシークレットを保存する
3. **あれば(2回目以降)**: `POST /v1/auth/login` にIDとシークレットを送り、アクセストークンを受け取る
4. 以降のAPIは `Authorization: Bearer <アクセストークン>` を付けて呼ぶ
5. トークンの期限(1時間)が切れて 401 が返ったら、手順3をやり直す

### API仕様

| メソッド・パス | 認証 | リクエスト | 成功時のレスポンス |
| --- | --- | --- | --- |
| `POST /v1/auth/guest` | 不要 | なし | 201。`user_id`, `login_secret`, `access_token`, `expires_at`(Unix秒) |
| `POST /v1/auth/login` | 不要 | `user_id`, `login_secret` | 200。`user_id`, `access_token`, `expires_at` |
| `GET /v1/me` | Bearer | なし | 200。`user_id`(動作確認用) |

エラーは `{"error": "invalid_credentials"}` の形で返す。IDの不一致とシークレットの不一致は同じ 401 にして、どちらが間違いかを外から判別できないようにする。

### DBスキーマ

```sql
-- migrations/001_init.sql
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at TIMESTAMPTZ
);

CREATE TABLE credentials (
    user_id     UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    secret_hash BYTEA NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### セキュリティ上の決めごと

- 通信は HTTPS のみにする(本番では Caddy や nginx を前段に置いて TLS を終端する)
- シークレットは 32 バイトの乱数(base64url 文字列)で、DBには SHA-256 のハッシュだけを保存する。パスワードと違って十分長い乱数なので、SHA-256 で足りる
- ハッシュの比較は `subtle.ConstantTimeCompare` を使う
- JWT の署名鍵は環境変数 `JWT_SECRET` で渡し、32 バイト以上のランダム値にする。コードやリポジトリには含めない
- クライアントの保存先は、開発中は PlayerPrefs で進め、公開前に iOS Keychain / Android Keystore を使う方式へ置き換える(PlayerPrefs は平文で保存されるため)
- ブルートフォース対策として、`/v1/auth/login` のIP単位のレート制限を後から追加する


### サーバー単体の動作確認

```bash
docker compose up -d
export DATABASE_URL='postgres://game:game@localhost:5432/game?sslmode=disable'
export JWT_SECRET="$(openssl rand -base64 32)"
go run ./cmd/api

# 別ターミナルで
curl -s -X POST localhost:8080/v1/auth/guest        # user_id と login_secret を控える
curl -s -X POST localhost:8080/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"<控えたID>","login_secret":"<控えたシークレット>"}'
curl -s localhost:8080/v1/me -H 'Authorization: Bearer <access_token>'
```

期待する結果: guest が 201、login が 200、`/v1/me` が `user_id` を返す。シークレットを1文字変えて login すると 401 になる。

### Unity側の動作確認

1. サーバーを起動した状態で再生する。初回は「ログイン成功」と `/v1/me -> 200` がConsoleに出る
2. 停止してもう一度再生する。ユーザーIDが前回と同じなら、保存した認証情報でのログインができている
3. 初回からやり直したいときは、メニューの Edit > Clear All PlayerPrefs で保存済みの認証情報を消す

**つまずきやすい点**: Android 実機は 9 以降で HTTP(暗号化なし)通信が既定で拒否される。ローカル接続の検証では Unity の Player 設定(Other Settings の Allow downloads over HTTP)を有効にするか、HTTPS のテストサーバーを使う。公開時は HTTPS に固定する。


### 通信の流れ(初回起動と2回目以降)

| 場面 | クライアントの動き | サーバーの動き |
| --- | --- | --- |
| 初回起動 | 保存なしと判断し `/v1/auth/guest` を呼ぶ → 返ったIDとシークレットを保存 | ユーザーと認証情報を作成し、トークンを返す |
| 2回目以降 | 保存済みの値で `/v1/auth/login` を呼ぶ | ハッシュを照合し、トークンを返す |
| 通常のAPI | `Authorization: Bearer` を付けて呼ぶ | ミドルウェアが検証し、ハンドラーへ渡す |
| 期限切れ(401) | `LoginAsync` を呼び直して再試行する | 新しいトークンを発行する |

