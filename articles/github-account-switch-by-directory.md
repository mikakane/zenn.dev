---
title: "GitHubアカウントをディレクトリごとに自動で切り替える（gh CLI + HTTPS）"
emoji: "🔀"
type: "tech"
topics: ["git", "github", "githubcli", "gh"]
published: false
---

## 目的

GitHub CLI（`gh`）を利用して、Gitリポジトリの配置ディレクトリに応じて、`git push` 時に使用するGitHubアカウントを自動的に切り替えます。

### 要件

- GitHubへの接続はHTTPSを使用する。
- `git remote` のURLは変更しない。
- 通常のディレクトリでは既存のデフォルトアカウントを使用する。
- 特定ディレクトリ配下では別のGitHubアカウントを使用する。
- `gh auth switch` による手動切り替えは不要とする。
- コミット作成者（`user.name` / `user.email`）も切り替え可能にする。

### ディレクトリ構成例

```text
~/projects/
├── app-a/              # デフォルトアカウント
├── app-b/              # デフォルトアカウント
└── work/
    ├── app-c/          # 仕事用アカウント
    └── app-d/          # 仕事用アカウント
```

## 前提条件

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

## 認証ヘルパースクリプトの作成

GitHub CLIから指定アカウントのトークンを取得するスクリプトを作成します。

### ディレクトリ作成

```bash
mkdir -p ~/.local/bin
```

### スクリプト作成

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

このスクリプトは、Gitから認証情報を要求された際に、指定されたGitHubアカウントのトークンを返します。

トークンはGit設定ファイルに保存されず、認証時にGitHub CLIから取得されます。

## デフォルトのGit設定

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
| `includeIf` | 特定ディレクトリ配下で追加設定を読み込む |

`includeIf` のパスは、実際の仕事用ディレクトリに合わせて変更してください。

:::message
既存の `~/.gitconfig` にCredential Helperの設定がある場合、重複や競合を避けるため設定を確認してください。デフォルトの認証が正常に動作している場合は、既存のCredential Helperをそのまま維持しても構いません。
:::

## 仕事用Git設定

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
| `YOUR_MAC_USER` | Macのログインユーザー名 |
| `work-user` | 仕事用GitHubユーザー名 |

Macのユーザー名は `whoami` で確認できます。

### Credential Helperの上書き

```ini
helper =
helper = "!/Users/YOUR_MAC_USER/.local/bin/git-gh-credential work-user"
```

最初の空の `helper =` は、それ以前に登録されたCredential Helperをリセットします。2行目で仕事用アカウント専用のヘルパーを登録します。

これにより、デフォルトの認証設定を仕事用ディレクトリ内だけで上書きできます。

## 動作確認

### デフォルトディレクトリ

```bash
cd ~/projects/app-a

git config --get user.email
git config --show-origin --get-all credential.helper
```

デフォルトのコミット作成者と認証設定が適用されていることを確認します。

### 仕事用ディレクトリ

```bash
cd ~/projects/work/app-c

git config --get user.email
git config --show-origin --get-all credential.https://github.com.helper
```

仕事用の設定が読み込まれていることを確認します。

### 実際の認証アカウントを確認

```bash
printf 'protocol=https\nhost=github.com\n\n' \
  | git credential fill \
  | sed -n '/^username=/p'
```

期待する出力：

```text
username=work-user
```

`sed` で `username` 行だけを抜き出しているので、認証トークン自体は表示されません。

### Pushテスト

```bash
git push
```

仕事用アカウントの権限で正常にpushできることを確認します。

## 運用上の注意

### GitHub CLIのアクティブアカウント

通常ディレクトリではGitHub CLIの標準認証ヘルパーが利用されるため、`gh auth switch` によってデフォルトの認証アカウントが変わる場合があります。

仕事用ディレクトリでは `gh auth token --user work-user` を使用するため、アクティブアカウントに依存しません。

### 環境変数

`GH_TOKEN` や `GITHUB_TOKEN` が設定されている場合、GitHub CLIが環境変数のトークンを優先する可能性があります。必要に応じて確認してください。

```bash
printenv GH_TOKEN >/dev/null && echo "GH_TOKEN is set"
printenv GITHUB_TOKEN >/dev/null && echo "GITHUB_TOKEN is set"
```

### Git設定の優先順位

リポジトリ固有の `.git/config` にCredential Helperが設定されている場合、追加のヘルパーが有効になることがあります。次のコマンドで設定元を確認できます。

```bash
git config --show-origin --get-all credential.helper
git config --show-origin --get-all credential.https://github.com.helper
```

### その他の注意

- 仕事用GitHubアカウントには対象リポジトリへの書き込み権限が必要です。
- OrganizationのSSO認可が必要な場合は、別途認可してください。
- `includeIf` はGitディレクトリの位置で判定されるため、Git worktreeなどでは追加設定が必要になる場合があります。
- この仕組みはGitのHTTPS認証に適用されます。`gh pr create` などGitHub CLI自体の操作アカウントは別途管理してください。

## まとめ

最終的な設定ファイル構成は以下のとおりです。

```text
~/
├── .gitconfig
│   └── デフォルト設定 + includeIf
│
├── .gitconfig-work
│   └── 仕事用Git設定
│
└── .local/
    └── bin/
        └── git-gh-credential
            └── GitHub CLI認証ヘルパー
```

| リポジトリ | 認証アカウント | コミット作成者 |
|---|---|---|
| `~/projects/app-a` | デフォルト | Default User |
| `~/projects/app-b` | デフォルト | Default User |
| `~/projects/work/app-c` | work-user | Work User |
| `~/projects/work/app-d` | work-user | Work User |

この構成では、Gitリポジトリを対象ディレクトリに配置するだけで、HTTPS認証アカウントとコミット作成者が自動的に切り替わります。既存のremote URLや各リポジトリの設定を変更する必要はありません。
