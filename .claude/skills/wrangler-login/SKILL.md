---
name: wrangler-login
description: Codespaces（リモート環境）でwrangler loginを行う。ブラウザを自動起動できない環境向けに、OAuth認証URLの提示・リダイレクトURLの受け取り・curlによるコールバック完了までを対話的に進める。「wranglerにログインしたい」「Cloudflareにログインできない」「wrangler loginがブラウザを開けない」といった相談で使う。
---

# wrangler-login: Codespaces上でのwrangler OAuthログイン

Codespaces（や他のリモート/ヘッドレス環境）では `wrangler login` がブラウザを自動起動できず、
認証後のリダイレクト先 `localhost` もユーザーの手元PCではなくリモート側を指すため、
そのままでは認証が完了しない。以下の手順で対話的にログインを完了させる。

参考: https://developers.cloudflare.com/workers/wrangler/commands/general/#use-wrangler-login-on-a-remote-machine

## 手順

### 1. 現在のログイン状態を確認する

```sh
npx wrangler whoami
```

すでにログイン済みならその旨をユーザーに伝え、再ログインが必要か確認する（不要ならここで終了）。

### 2. `npx wrangler login` をバックグラウンドで起動する

- 必ずバックグラウンド実行にする（フォアグラウンドで実行するとコールバックを待つ間ブロックしてしまう）
- 起動直後は認証URLがまだ出力されていないことがあるため、出力ファイルを数秒おきに再確認する
- 出力から `Please visit the following URL in your browser:` の下のURLを抽出する

### 3. 認証URLをユーザーに提示する

- URLをそのまま貼り、手元PCのブラウザで開いて認証・認可してもらうよう案内する
- 認証後 `http://localhost:8976/oauth/callback?code=...&state=...` へリダイレクトされ、
  ブラウザ上ではページが表示されない（接続エラーになる）のが正常であることを伝える
- **リダイレクト後のアドレスバーの完全なURL**をそのまま貼ってもらうよう依頼する

### 4. ユーザーから受け取ったURLをcurlする

```sh
curl -s "<ユーザーから受け取った完全なlocalhost URL>"
```

- URLは `&` や `~` を含むため必ずダブルクォートで囲む
- 受け取った`state`パラメータが手順3で提示したURLの`state`と一致しているか確認してからcurlする
  （一致しない場合、古いセッションの使い回しの可能性があるため手順2からやり直す）

### 5. バックグラウンドプロセスの出力を確認する

- `Successfully logged in.` が出ていれば成功
- `Timed out waiting for authorization code` が出ている場合、認証が間に合わなかったということなので
  手順2からやり直す（wrangler loginのタイムアウトは数分程度）

### 6. `npx wrangler whoami` で最終確認する

アカウント名・メールアドレスが表示されればログイン完了。結果をユーザーに報告する。

## 注意点

- 古いセッションでブラウザ認証だけ完了していても、対応する`wrangler login`プロセスがタイムアウト等で
  終了済みの場合、PKCEの`code_verifier`が失われているためcurlしても失敗する。必ず手順2〜4は
  同じバックグラウンドプロセスが生きている間に一連で行うこと。
- コンテナ環境固有の`--callback-host=0.0.0.0`オプションはCodespaces（通常のリモートマシン扱い）では不要。
