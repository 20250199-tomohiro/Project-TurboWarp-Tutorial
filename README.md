# TurboWarp ミラーを Render にデプロイする手順

TurboWarp/mirror のコードを使い、Render 上に TurboWarp のミラーを作るまでの手順と、
途中で出た問題の対処をまとめたものです。

## 全体の構成

ミラーを正しく動かすには、Render 上に次の3つのサービスが必要でした。

| サービス | 種類 | 役割 |
| --- | --- | --- |
| ミラー本体 | Static Site | TurboWarp のエディタ(scratch-gui)をビルドして配信する |
| trampoline | Web Service | Scratch のプロジェクト情報を取得するプロキシ(自前でホストする) |
| 拡張機能ギャラリー | Static Site | TurboWarp の拡張機能ギャラリー(自前でホストする) |

TurboWarp/mirror の README は GitHub Pages 向けの手順しか書かれていないため、
Render では `build.sh` を自作してビルドします。
`patch.py` は、`scratch-gui/build` の中のファイルを加工する(noindex の追加、不要ファイルの削除、
editor.html を index.html にコピー、robots.txt の配置)ためのスクリプトです。

## 事前準備

- GitHub アカウントと Render アカウント
- 以下のリポジトリを自分のアカウントにフォークしておく(Git コマンドは不要。GitHub の画面だけで可能)
  - `TurboWarp/mirror`(ミラー本体)
  - `TurboWarp/trampoline`

## 手順1: trampoline をデプロイする

Render で **New + → Web Service** を選び、フォークした trampoline のリポジトリを接続します。

| 項目 | 値 |
| --- | --- |
| Build Command | `npm ci` |
| Start Command | `node src/index.js` |

- `npm start` は使えません(`package.json` に start スクリプトがないため)。
- ポートは Render が渡す `PORT` 環境変数を読む作りなので、設定は不要です。
- 許可オリジンは `*`(全許可)です。
- デプロイ後、次の URL をブラウザで開いて JSON(`project_token` を含む)が表示されれば成功です。

```
https://あなたのtrampoline.onrender.com/api/projects/1379737786
```

### 注意

- Render の無料プランでは、しばらくアクセスがないとスリープします。スリープ直後の最初の読み込みは、
  遅くなったり失敗したりすることがあります。少し待って再読み込みしてください。
- trampoline は、Scratch からの取得に失敗した結果も 30 分間キャッシュします。
  一度失敗すると、しばらく同じエラーが返ることがあります。
  無料プランではキャッシュ(SQLite ファイル)は再デプロイで消えます。
- 公開されている trampoline のコードは、本家の本番版とは異なる部分があります
  (リポジトリの README に記載あり)。

## 手順2: 拡張機能ギャラリーをデプロイする

TurboWarp の公式ギャラリーは `extensions.turbowarp.org` にありますが、
環境によってはこのドメインにアクセスできないことがあります。
その場合は、`TurboWarp/extensions` を自分でホストします。

Render で **New + → Static Site** を選び、`https://github.com/TurboWarp/extensions` を接続します。

| 項目 | 値 |
| --- | --- |
| Build Command | `npm ci && npm run build` |
| Publish Directory | `build` |

(この2つの値は推測で案内したものです。実際に動いた設定がこれと違う場合は、書き換えてください。)

さらに、ミラーから取得できるように CORS ヘッダーを追加します。

1. そのサービスの **Headers**(見つからなければ Settings 内)を開く
2. **Add Header** で次を追加して保存する

| 項目 | 値 |
| --- | --- |
| Path | `/*` |
| Name | `Access-Control-Allow-Origin` |
| Value | `*` |

## 手順3: ミラー本体をデプロイする

### 3-1. リポジトリに `build.sh` を追加する

フォークしたミラーのリポジトリに、GitHub の画面から「Add file → Create new file」で
`build.sh` を作成し、次の内容を貼り付けます。

```bash
#!/usr/bin/env bash
set -euo pipefail
: "${TRAMPOLINE_HOST:?TRAMPOLINE_HOST を設定してください}"
: "${EXTENSIONS_HOST:?EXTENSIONS_HOST を設定してください}"

# 前回のビルドのキャッシュが残っていると git clone が失敗するため、先に削除する
rm -rf scratch-gui
git clone --depth 1 https://github.com/TurboWarp/scratch-gui
cd scratch-gui
npm ci

# trampoline の URL が書かれたファイルを探して、自前の trampoline に置換する
grep -rlI "trampoline\.turbowarp\." src node_modules/@turbowarp node_modules/scratch-* > /tmp/hits.txt 2>/dev/null || true
echo "--- 置換対象 ---"
cat /tmp/hits.txt
[ -s /tmp/hits.txt ] || { echo "trampolineのURLが見つかりませんでした"; exit 1; }
xargs -a /tmp/hits.txt sed -i -E "s#trampoline\.turbowarp\.(xyz|org)#${TRAMPOLINE_HOST}#g"

# 拡張機能ギャラリーの取得先を、自前のギャラリーに置換する
sed -i "s#extensions\.turbowarp\.org#${EXTENSIONS_HOST}#g" \
  src/containers/extension-library.jsx \
  src/lib/libraries/extensions/index.jsx \
  src/containers/tw-security-manager.jsx

NODE_ENV=production npm run build
cd ..
python3 patch.py
```

置換している箇所:

- `src/lib/tw-project-meta-fetcher-hoc.jsx`: プロジェクト情報を trampoline から取得する部分
  (`.org` を先に試し、失敗したら `.xyz` を試す作りだった)
- `src/containers/extension-library.jsx`: 拡張機能ギャラリーの一覧と、各拡張機能の取得先
- `src/containers/tw-security-manager.jsx`: 自前のギャラリーの拡張機能を、警告なしで読み込めるようにする

### 3-2. Render で Static Site を作る

Render で **New + → Static Site** を選び、フォークしたミラーのリポジトリを接続します。

| 項目 | 値 |
| --- | --- |
| Build Command | `bash build.sh` |
| Publish Directory | `scratch-gui/build` |

**Environment**(環境変数)に次の2つを追加します。値はドメインだけを書きます
(`https://` や末尾の `/` は付けない)。

| 名前 | 値の例 |
| --- | --- |
| `TRAMPOLINE_HOST` | `あなたのtrampoline.onrender.com` |
| `EXTENSIONS_HOST` | `あなたのextensions.onrender.com` |

作成後、**Clear build cache & deploy** を実行します。

### 3-3. ビルドログで確認する

ビルドログの「--- 置換対象 ---」の下に、ファイル名が出ていれば置換は成功です
(今回は次の5ファイルが出ていた)。

```
src/components/tw-studioview/studioview.js
src/lib/tw-project-meta-fetcher-hoc.jsx
src/playground/credits/users.js
node_modules/scratch-vm/src/playground/benchmark.js
node_modules/scratch-vm/src/extensions/scratch3_translate/index.js
```

最後に「Your service is live」と表示されればデプロイ完了です。

## 動作確認

- ミラーの URL の末尾に `#プロジェクトID` を付けて開き、プロジェクトが読み込めること。
- 拡張機能の追加画面で、ギャラリーが表示されること。
- 古いキャッシュが残っていると、修正前のコードが動くことがあります。
  シークレットウィンドウが使えない場合は、**Ctrl+Shift+R**(強制再読み込み)、
  URL に `?v=2` のような文字列を付ける、サイトデータの削除、のいずれかを試してください。

## トラブルシューティング

| 症状 | 原因と対処 |
| --- | --- |
| `fatal: destination path 'scratch-gui' already exists` | 前回のビルドのキャッシュが残っている。`build.sh` の先頭で `rm -rf scratch-gui` を実行する。 |
| `npm error Missing script: "start"`(trampoline) | start スクリプトがない。Start Command を `node src/index.js` にする。 |
| `Failed to fetch https://trampoline.turbowarp.xyz/api/projects/...` | 公式の trampoline に届いていない。trampoline を自前でホストし、`build.sh` で URL を置換する。 |
| 「このプロジェクトは未共有」と表示されるが、公式サイトでは読み込める | 自前の trampoline を直接開いて JSON が返るか確認する。返るなら、古いキャッシュを疑う(強制再読み込み)。失敗結果は trampoline 側でも 30 分キャッシュされる。 |
| 拡張機能ギャラリーが読み込めない | `extensions.turbowarp.org` にアクセスできない可能性がある。ギャラリーを自前でホストし、`EXTENSIONS_HOST` で置換する。CORS ヘッダーの設定も必要。 |
| 「trampolineのURLが見つかりませんでした」でビルドが止まる | scratch-gui の更新で URL の書き方が変わった可能性がある。ビルドログを確認して、検索範囲を直す。 |
| ビルド中に `JavaScript heap out of memory` | 環境変数に `NODE_OPTIONS=--max-old-space-size=4096` を追加する。 |

## デバッグのヒント

- Render の Start Command を `cat src/api.js` のように一時的に変えると、サーバーを起動せず
  ファイルの内容を Logs に表示できる。確認後は元のコマンドに戻すこと。
- trampoline の環境変数に `NODE_ENV=development` を追加すると、デバッグログが出る。
  確認後は削除すること。
- ブラウザの開発者ツールが使えない環境でも、Render の Logs とビルドログだけで原因を絞り込める。

## 既知の制限と注意点

- 無料プランの Web Service はスリープする。最初の読み込みが遅くなることがある。
- trampoline の許可オリジンは `*` のため、URL を知っていれば誰でも利用できる。
  利用が増える場合は、Render の無料枠を超えないか確認すること。
- `build.sh` は scratch-gui の最新版をビルドのたびに clone する。
  ビルドが急に失敗するようになったら、ログを確認すること。
- 本家のミラーは約1週間ごとに自動更新される。Render では自動更新されないため、
  必要なら Manual Deploy(Clear build cache & deploy)を実行する。
  自動で行いたい場合は、Render の Deploy Hook を GitHub Actions の定期実行から呼び出す方法がある。

## 参考

- TurboWarp/mirror: https://github.com/TurboWarp/mirror
- TurboWarp/trampoline: https://github.com/TurboWarp/trampoline
- TurboWarp/extensions: https://github.com/TurboWarp/extensions
- TurboWarp 開発ドキュメント: https://docs.turbowarp.org/development/getting-started
- 未共有プロジェクトについて: https://docs.turbowarp.org/unshared-projects
