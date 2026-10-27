---
title: "GitHubアカウントをディレクトリごとに自動で切り替える（gh CLI + HTTPS）"
emoji: "🔀"
type: "tech"
topics: ["git", "github", "githubcli", "gh"]
published: false
---

## 1. 構成の概要

GitHub CLI（`gh`）を利用して、Gitリポジトリの配置ディレクトリに応じて、`git push` 時に使用するGitHubアカウントを自動的に切り替えます。

### 1-1. 要件

- GitHubへの接続はHTTPSを使用する。
- `git remote` のURLは変更しない。
- 通常のディレクトリでは既存のデフォルトアカウントを使用する。
- 特定ディレクトリ（以下「除外フォルダ」）配下では別のGitHubアカウントを使用する。
- `gh auth switch` による手動切り替えは不要とする。
- コミット作成者（`user.name` / `user.email`）も切り替え可能にする。

### 1-2. 仕組み

```text
~/
├── .gitconfig                # デフォルト設定 + includeIf
├── .gitconfig-work           # 除外フォルダ用の設定（仕事用アカウント）
└── .local/
    └── bin/
        └── git-gh-credential # GitHub CLI からトークンを取得する認証ヘルパー
```

| 要素 | 役割 |
|---|---|
| `~/.gitconfig` | デフォルトのコミット作成者と認証（`gh auth git-credential`）を定義し、`includeIf` で除外フォルダ用の設定を読み込む |
| `~/.gitconfig-work` | 除外フォルダ配下でコミット作成者とCredential Helperを上書きする |
| `git-gh-credential` | 指定アカウントのトークンを `gh auth token --user` で取得してGitに返す |

トークンはGit設定ファイルに保存されず、認証のたびにGitHub CLIから取得されます。

### 1-3. ディレクトリ構成例と動作

```text
~/projects/
├── app-a/              # デフォルトアカウント
├── app-b/              # デフォルトアカウント
└── work/               # 除外フォルダ
    ├── app-c/          # 仕事用アカウント
    └── app-d/          # 仕事用アカウント
```

| リポジトリ | 認証アカウント | コミット作成者 |
|---|---|---|
| `~/projects/app-a` | デフォルト | Default User |
| `~/projects/app-b` | デフォルト | Default User |
| `~/projects/work/app-c` | work-user | Work User |
| `~/projects/work/app-d` | work-user | Work User |

Gitリポジトリを除外フォルダに配置するだけで、HTTPS認証アカウントとコミット作成者が自動的に切り替わります。既存のremote URLや各リポジトリの設定を変更する必要はありません。

## 2. 初回セットアップ

### 2-1. 前提条件

- Gitがインストールされている。
- GitHub CLI（`gh`）がインストールされている。
- GitHub CLIにデフォルト用・仕事用の両アカウントが登録されている。

ログイン状態を確認します。

```bash
gh auth status
```

未登録のアカウントがあれば追加します。

```bash
gh auth login --git-protocol https
```

仕事用アカウントのトークンが取得できることを確認します。

```bash
gh auth token --hostname github.com --user work-user > /dev/null
```

`work-user` は実際のGitHubユーザー名に置き換えてください。

### 2-2. 認証ヘルパースクリプトの作成

GitHub CLIから指定アカウントのトークンを取得するスクリプトを作成します。

```bash
mkdir -p ~/.local/bin
```

`~/.local/bin/git-gh-credential` を作成します。

```bash
cat > ~/.local/bin/git-gh-credential <<'EOF'
#!/bin/sh

account="$1"
operation="$2"

[ "$operation" = "get" ] || exit 0

host=""
while IFS= read -r line; do
    case "$line" in
        host=*) host="${line#host=}" ;;
        "") break ;;
    esac
done

[ "$host" = "github.com" ] || exit 0

token=$(gh auth token --hostname github.com --user "$account") || exit 1

printf 'username=%s\n' "$account"
printf 'password=%s\n' "$token"
EOF
```

実行権限を付与します。

```bash
chmod 700 ~/.local/bin/git-gh-credential
```

このスクリプトは、Gitから認証情報を要求された際に、第1引数で指定されたGitHubアカウントのトークンを返します。

### 2-3. デフォルトのGit設定

`~/.gitconfig` を編集します。既存の設定は維持し、必要な設定のみ追加します。

```ini:~/.gitconfig
[user]
    name = Default User
    email = default@example.com

[credential "https://github.com"]
    helper =
    helper = !gh auth git-credential

[includeIf "gitdir:~/projects/work/"]
    path = ~/.gitconfig-work
```

| 設定 | 説明 |
|---|---|
| `[user]` | デフォルトのコミット作成者 |
| `credential.helper` | デフォルトのGitHub HTTPS認証 |
| `includeIf` | 除外フォルダ配下で追加設定を読み込む |

`includeIf` のパスは、実際の除外フォルダに合わせて変更してください。

:::message
既存の `~/.gitconfig` にCredential Helperの設定がある場合、重複や競合を避けるため設定を確認してください。デフォルトの認証が正常に動作している場合は、既存のCredential Helperをそのまま維持しても構いません。
:::

### 2-4. 除外フォルダ用のGit設定

`~/.gitconfig-work` を作成します。

```bash
touch ~/.gitconfig-work
```

以下を記述します。

```ini:~/.gitconfig-work
[user]
    name = Work User
    email = work@example.com

[credential "https://github.com"]
    helper =
    helper = "!/Users/YOUR_MAC_USER/.local/bin/git-gh-credential work-user"
```

以下を実際の値に置き換えてください。

| 項目 | 設定値 |
|---|---|
| `Work User` | 仕事用のコミット作成者名 |
| `work@example.com` | 仕事用メールアドレス |
| `YOUR_MAC_USER` | Macのログインユーザー名（`whoami` で確認） |
| `work-user` | 仕事用GitHubユーザー名 |

最初の空の `helper =` は、それ以前に登録されたCredential Helperをリセットします。2行目で仕事用アカウント専用のヘルパーを登録します。これにより、デフォルトの認証設定を除外フォルダ内だけで上書きできます。

### 2-5. 動作確認

デフォルトディレクトリで、デフォルトのコミット作成者と認証設定が適用されていることを確認します。

```bash
cd ~/projects/app-a

git config --get user.email
git config --show-origin --get-all credential.helper
```

除外フォルダで、仕事用の設定が読み込まれていることを確認します。

```bash
cd ~/projects/work/app-c

git config --get user.email
git config --show-origin --get-all credential.https://github.com.helper
```

実際に使われる認証アカウントを確認します。`sed` で `username` 行だけを抜き出しているので、トークン自体は表示されません。

```bash
printf 'protocol=https\nhost=github.com\n\n' \
  | git credential fill \
  | sed -n '/^username=/p'
```

期待する出力：

```text
username=work-user
```

最後に `git push` を実行し、仕事用アカウントの権限で正常にpushできることを確認します。

```bash
git push
```

## 3. 除外フォルダの追加

初回セットアップ後に、別のアカウントを使うフォルダを増やす手順です。

### 3-1. 既存アカウントを使うフォルダを追加する

同じ仕事用アカウントを別のディレクトリでも使う場合は、`~/.gitconfig` に `includeIf` を追加するだけです。

```ini:~/.gitconfig
[includeIf "gitdir:~/projects/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/projects/client-x/"]
    path = ~/.gitconfig-work
```

:::message
`gitdir:` のパスは末尾を `/` にしてください。末尾が `/` の場合は配下すべてのリポジトリが対象になります（`gitdir:~/projects/work/**` と同じ扱い）。
:::

### 3-2. 別のアカウントを使うフォルダを追加する

3つ目以降のアカウントを使う場合は、アカウントごとに設定ファイルを分けます。

まずGitHub CLIにアカウントを追加し、トークンが取得できることを確認します。

```bash
gh auth login --git-protocol https
gh auth token --hostname github.com --user other-user > /dev/null
```

`~/.gitconfig-other` を作成します。認証ヘルパースクリプトは共通のものを使い、引数のアカウント名だけを変えます。

```ini:~/.gitconfig-other
[user]
    name = Other User
    email = other@example.com

[credential "https://github.com"]
    helper =
    helper = "!/Users/YOUR_MAC_USER/.local/bin/git-gh-credential other-user"
```

`~/.gitconfig` に `includeIf` を追加します。

```ini:~/.gitconfig
[includeIf "gitdir:~/projects/oss/"]
    path = ~/.gitconfig-other
```

### 3-3. 追加後の確認

追加したフォルダ配下のリポジトリで、「2-5. 動作確認」 と同じコマンドを実行します。`username=` に期待するアカウント名が表示されれば完了です。

```bash
cd ~/projects/oss/some-repo

git config --get user.email
printf 'protocol=https\nhost=github.com\n\n' \
  | git credential fill \
  | sed -n '/^username=/p'
```

## 4. 運用上の注意

### 4-1. GitHub CLIのアクティブアカウント

通常ディレクトリではGitHub CLIの標準認証ヘルパーが利用されるため、`gh auth switch` によってデフォルトの認証アカウントが変わる場合があります。

除外フォルダでは `gh auth token --user <アカウント>` を使用するため、アクティブアカウントに依存しません。

### 4-2. 環境変数

`GH_TOKEN` や `GITHUB_TOKEN` が設定されている場合、GitHub CLIが環境変数のトークンを優先する可能性があります。必要に応じて確認してください。

```bash
printenv GH_TOKEN >/dev/null && echo "GH_TOKEN is set"
printenv GITHUB_TOKEN >/dev/null && echo "GITHUB_TOKEN is set"
```

### 4-3. Git設定の優先順位

リポジトリ固有の `.git/config` にCredential Helperが設定されている場合、追加のヘルパーが有効になることがあります。次のコマンドで設定元を確認できます。

```bash
git config --show-origin --get-all credential.helper
git config --show-origin --get-all credential.https://github.com.helper
```

### 4-4. その他の注意

- 除外フォルダで使うGitHubアカウントには、対象リポジトリへの書き込み権限が必要です。
- OrganizationのSSO認可が必要な場合は、別途認可してください。
- `includeIf` はGitディレクトリの位置で判定されるため、Git worktreeなどでは追加設定が必要になる場合があります。
- この仕組みはGitのHTTPS認証に適用されます。`gh pr create` などGitHub CLI自体の操作アカウントは別途管理してください。

## 5. この記事をベースに作業を依頼されているAIへ

この記事を参照してユーザーの設定作業を代行・支援する場合は、以下の手順に従ってください。

### 5-1. 作業種別の確認

作業を始める前に、ユーザーに次のどちらかを確認してください。推測で進めないでください。

| 作業種別 | 内容 | 参照する章 |
|---|---|---|
| 新規 | この仕組みをまだ導入していない環境に、初めてセットアップする | 1章、2章（2-1〜2-5）、4章 |
| 追加 | 導入済みの環境に、除外フォルダを追加する | 1章、3章（3-1〜3-3）、4章 |

判断がつかない場合は、`~/.local/bin/git-gh-credential` の有無と、`~/.gitconfig` に `includeIf` があるかを確認し、その結果をユーザーに提示したうえで判断を仰いでください。

```bash
ls -l ~/.local/bin/git-gh-credential
git config --global --get-regexp '^includeif\.'
```

### 5-2. 「新規」の場合の確認項目

ユーザーに以下を確認してから、2章の手順を順に実施してください。

| 確認項目 | 用途 | 参照する章 |
|---|---|---|
| デフォルトで使うGitHubアカウント名 | `gh auth status` での登録確認 | 2-1 |
| 除外フォルダで使うGitHubアカウント名 | `git-gh-credential` の引数 | 2-1, 2-4 |
| 両アカウントが `gh` に登録済みか | 未登録なら `gh auth login` をユーザー自身に実行してもらう | 2-1 |
| デフォルトのコミット作成者名・メールアドレス | `~/.gitconfig` の `[user]` | 2-3 |
| 除外フォルダのパス | `includeIf "gitdir:..."` | 2-3 |
| 除外フォルダ用のコミット作成者名・メールアドレス | `~/.gitconfig-work` の `[user]` | 2-4 |
| 既存の `~/.gitconfig` のCredential Helper設定 | 既存設定と競合しないか | 2-3 |
| `GH_TOKEN` / `GITHUB_TOKEN` が設定されていないか | 意図しないトークンが使われないか | 4-2 |

完了条件：2-5の確認で、除外フォルダ配下の `username=` が期待するアカウント名になっていること。

### 5-3. 「追加」の場合の確認項目

まず、追加するフォルダで使うアカウントが既存の設定ファイル（例：`~/.gitconfig-work`）と同じか、新しいアカウントかを確認し、3-1と3-2のどちらで進めるかを決めてください。

| 確認項目 | 用途 | 参照する章 |
|---|---|---|
| 追加するフォルダのパス | `includeIf "gitdir:..."`（末尾は `/`） | 3-1, 3-2 |
| 既存アカウントか新規アカウントか | 3-1と3-2のどちらで進めるか | 3-1, 3-2 |
| （既存アカウントの場合）流用する設定ファイル名 | `includeIf` の `path` | 3-1 |
| （新規アカウントの場合）GitHubアカウント名 | `gh auth token --user` と `git-gh-credential` の引数 | 3-2 |
| （新規アカウントの場合）アカウントが `gh` に登録済みか | 未登録なら `gh auth login` をユーザー自身に実行してもらう | 3-2 |
| （新規アカウントの場合）コミット作成者名・メールアドレス | 新しい設定ファイルの `[user]` | 3-2 |
| 既存の `includeIf` とパスが重複していないか | 意図しない設定の上書きを防ぐ | 3-1, 3-2 |

完了条件：3-3の確認で、追加したフォルダ配下の `username=` が期待するアカウント名になっていること。

### 5-4. 共通の注意

- トークンの値を画面やログに出力しないでください。確認には2-5の `sed -n '/^username=/p'` のように `username` 行だけを表示する方法を使ってください。
- `gh auth login` はブラウザ認証を伴うため、ユーザー自身に実行してもらってください。
- `~/.gitconfig` を編集する前に、現在の内容をユーザーに提示し、変更点を説明してから反映してください。
