# ipa-download

App Store のアプリURLを指定して ipa ファイルをダウンロードする GitHub Action。
内部で [`ipatool`](https://github.com/majd/ipatool)（非公式ツール）を使用します。

## できること / できないこと

- ✅ **自分の Apple ID でライセンスを持っている（無料アプリ・購入済みの有料アプリ）** の ipa を、App Store の URL から取得できます
- ❌ **有料アプリの自動購入(課金)はしません**。`--purchase` は無料アプリの「ライセンス取得」のみを行います
- ❌ **完全な放置運用はできません**。Apple の2段階認証(2FA)セッションには有効期限があり、切れたら下記「初回セットアップ」を再実行してシークレットを更新する必要があります
- ⚠️ `ipatool` は Apple 非公式の API を利用しています。個人の検証目的を超えた大量ダウンロード・再配布はApple利用規約違反およびアカウント停止のリスクがあります。自己責任で利用してください

## 仕組み

Apple は未知の環境からのサインインに毎回2FAを要求するため、GitHub Actions の使い捨て環境内で2FA自体を突破することはできません。
そのため、**2FA突破は手元の端末で一度だけ行い、その結果できた認証セッション（`ipatool`のfile-basedキーリング一式）をGitHub Secretsに保存して使い回す**設計にしています。

```
[あなたの手元PC]                          [GitHub Actions]
  ipatool auth login (2FAを1回入力)
       │
       ▼
  ~/.ipatool/ (セッション一式)
       │ base64化してSecretsに登録
       ▼
  IPATOOL_SESSION_B64 (Secret)   ───────▶  セッションを復元して
  IPATOOL_KEYCHAIN_PASSPHRASE(Secret) ──▶  ipatool download を実行
```

## 初回セットアップ（手元の端末で1回だけ）

### 1. ipatool をインストール

- macOS: `brew install ipatool`
- Linux/Windows: [Releases](https://github.com/majd/ipatool/releases) から自分のOS用バイナリを取得

### 2. ログイン（2FAコードが必要）

キーリングを暗号化するための任意のパスフレーズを決めて、ログインします。

```bash
export APPLE_ID="your-apple-id@example.com"
export APPLE_PASSWORD="your-apple-password"
export IPATOOL_KEYCHAIN_PASSPHRASE="好きなパスフレーズを決めてください"

ipatool auth login -e "$APPLE_ID" -p "$APPLE_PASSWORD" --keychain-passphrase "$IPATOOL_KEYCHAIN_PASSPHRASE"
```

- iPhoneに届いた6桁コードを聞かれたら入力してください
- もし `--auth-code` を明示的に渡す必要がある場合は以下のように実行してください
  ```bash
  ipatool auth login -e "$APPLE_ID" -p "$APPLE_PASSWORD" --auth-code 123456 --keychain-passphrase "$IPATOOL_KEYCHAIN_PASSPHRASE"
  ```

成功すると `~/.ipatool/` にセッション情報（暗号化済み）が保存されます。

### 3. セッションを取り出して base64 化

```bash
tar czf ipatool-session.tar.gz -C "$HOME" .ipatool
base64 -w0 ipatool-session.tar.gz > ipatool-session.b64
```

`ipatool-session.b64` の中身をコピーします。

### 4. GitHub Secrets に登録

リポジトリの **Settings → Secrets and variables → Actions** で以下を登録してください。

| Secret名 | 値 |
|---|---|
| `IPATOOL_SESSION_B64` | 手順3で作った `ipatool-session.b64` の中身全体 |
| `IPATOOL_KEYCHAIN_PASSPHRASE` | 手順2で決めたパスフレーズ |

`APPLE_ID` / `APPLE_PASSWORD` は今回のワークフローでは直接使用しません（セッションのみで動作するため）。ただしセッション更新時に再度必要になるので保存したままで問題ありません。

## 使い方

1. GitHub の **Actions タブ → "Download IPA from App Store URL"** を開く
2. **Run workflow** をクリック
3. `app_store_url` に App Store のアプリURL（例: `https://apps.apple.com/jp/app/xxxx/id123456789`）を入力して実行
4. 完了後、ワークフローの Artifacts から `.ipa` をダウンロード

## セッションが切れたら

`download` ステップが認証エラーで失敗するようになったら、上記「初回セットアップ」の手順2〜4を再実行してシークレットを更新してください。

## 免責

- 本ツールは非公式APIに依存しており、Appleの仕様変更で突然動かなくなる可能性があります
- ダウンロードしたipaの取り扱い（再配布・改変等）は各アプリの利用規約・著作権法に従ってください
