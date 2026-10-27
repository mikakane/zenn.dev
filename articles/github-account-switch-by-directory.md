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

:::message
2章ではデフォルトアカウントでの認証と認証ヘルパーの準備までを行い、除外フォルダは設定しません。除外フォルダは3章の手順で追加します。
:::

### 2-1. 前提条件

- Gitがインストールされている。
- GitHub CLI（`gh`）がインストールされている。
- 使用するGitHubアカウントがすべて GitHub CLI にログイン済みである。

ログイン状態を確認します。

```bash
gh auth status
```

未登録のアカウントがあれば追加します。

```bash
gh auth login --git-protocol https
```

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

このスクリプトは、Gitから認証情報を要求された際に、第1引数で指定されたGitHubアカウントのトークンを返します。3章で除外フォルダを追加するときに使います。

### 2-3. デフォルトのGit設定

`~/.gitconfig` を編集します。既存の設定は維持し、必要な設定のみ追加します。

```ini:~/.gitconfig
[user]
    name = Default User
    email = default@example.com

[credential "https://github.com"]
    helper =
    helper = !gh auth git-credential
```

| 設定 | 説明 |
|---|---|
| `[user]` | デフォルトのコミット作成者 |
| `credential.helper` | デフォルトのGitHub HTTPS認証 |

:::message
既存の `~/.gitconfig` にCredential Helperの設定がある場合、重複や競合を避けるため設定を確認してください。デフォルトの認証が正常に動作している場合は、既存のCredential Helperをそのまま維持しても構いません。
:::

### 2-4. 動作確認

任意のリポジトリで、デフォルトのコミット作成者と認証設定が適用されていることを確認します。

```bash
cd ~/projects/app-a

git config --get user.email
git config --show-origin --get-all credential.https://github.com.helper
```

## 3. 除外フォルダの追加

別のアカウントを使うフォルダを追加する手順です。最初の除外フォルダも、2つ目以降もこの章の手順で追加します。

除外フォルダで使うアカウントが、すでに作成済みの設定ファイル（例：`~/.gitconfig-work`）と同じアカウントであれば、3-1を飛ばして3-2に進みます。

### 3-1. アカウント用の設定ファイルを作成する

除外フォルダで使うアカウントのトークンが取得できることを確認します。

```bash
gh auth token --hostname github.com --user work-user > /dev/null
```

アカウント用の設定ファイル（ここでは `~/.gitconfig-work`）を作成し、以下を記述します。

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
| `Work User` | 除外フォルダ用のコミット作成者名 |
| `work@example.com` | 除外フォルダ用のメールアドレス |
| `YOUR_MAC_USER` | Macのログインユーザー名（`whoami` で確認） |
| `work-user` | 除外フォルダで使うGitHubユーザー名 |

最初の空の `helper =` は、それ以前に登録されたCredential Helperをリセットします。2行目でこのアカウント専用のヘルパーを登録します。これにより、デフォルトの認証設定を除外フォルダ内だけで上書きできます。

アカウントを増やす場合は、`~/.gitconfig-other` のようにアカウントごとに設定ファイルを分けます。認証ヘルパースクリプトは共通のものを使い、引数のアカウント名だけを変えます。

### 3-2. includeIf を追加する

`~/.gitconfig` に `includeIf` を追加し、除外フォルダ配下で 3-1 の設定ファイルを読み込むようにします。

```ini:~/.gitconfig
[includeIf "gitdir:~/projects/work/"]
    path = ~/.gitconfig-work
```

同じアカウントを別のフォルダでも使う場合は、同じ設定ファイルを指す `includeIf` を追加するだけです。

```ini:~/.gitconfig
[includeIf "gitdir:~/projects/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/projects/client-x/"]
    path = ~/.gitconfig-work
```

:::message
`gitdir:` のパスは末尾を `/` にしてください。末尾が `/` の場合は配下すべてのリポジトリが対象になります（`gitdir:~/projects/work/**` と同じ扱い）。
:::

### 3-3. 動作確認

除外フォルダ配下のリポジトリで、除外フォルダ用の設定が読み込まれていることを確認します。

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

最後に `git push` を実行し、除外フォルダ用アカウントの権限で正常にpushできることを確認します。

```bash
git push
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

使用するGitHubアカウントは、すべて GitHub CLI にログイン済みであることを前提とします。ログイン状態はユーザーに尋ねず、AIが `gh auth status` で確認してください。必要なアカウントが未登録の場合は作業を止め、ユーザー自身に `gh auth login --git-protocol https` を実行してもらってください。

ユーザーへの確認は、ユーザーにしか決められない項目に絞ります。コマンドで調べられることはAIが確認し、その結果をユーザーへの確認時に添えてください。

### 5-1. 作業種別の判定

AIが次のコマンドで導入状況を確認し、作業種別を判定してください。

```bash
ls -l ~/.local/bin/git-gh-credential
git config --global --get-regexp '^includeif\.'
```

| 作業種別 | 判定の目安 | 参照する章 |
|---|---|---|
| 新規 | `git-gh-credential` がない | 1章、2章、3章、4章 |
| 追加 | `git-gh-credential` があり、2章の設定が済んでいる | 1章、3章、4章 |

判定結果をユーザーに提示し、作業種別に誤りがないか確認してから進めてください。「新規」の場合は、2章を終えたあとに続けて3章で除外フォルダを追加します。

### 5-2. 「新規」の場合

#### ユーザーへの確認項目

| 確認項目 | 用途 | 参照する章 |
|---|---|---|
| デフォルトで使うGitHubアカウント | デフォルト認証の確認 | 2-1 |
| デフォルトのコミット作成者名・メールアドレス | `~/.gitconfig` の `[user]` | 2-3 |

アカウントは `gh auth status` に表示された一覧から選んでもらってください。コミット作成者は、既存の `~/.gitconfig` に設定があればその値を提示し、そのまま使うか確認してください。除外フォルダの情報は、2章を終えたあとに 5-3 の手順で確認します。

#### AIの確認項目

| 確認項目 | 確認方法 | 参照する章 |
|---|---|---|
| アカウントが `gh` にログイン済みか | `gh auth status` | 2-1 |
| 既存のコミット作成者 | `git config --global --get user.name` / `git config --global --get user.email` | 2-3 |
| 既存のCredential Helper設定 | `git config --global --show-origin --get-all credential.helper` / `git config --global --show-origin --get-all credential.https://github.com.helper` | 2-3 |
| `GH_TOKEN` / `GITHUB_TOKEN` が設定されていないか | `printenv GH_TOKEN >/dev/null && echo set` / `printenv GITHUB_TOKEN >/dev/null && echo set` | 4-2 |

完了条件：2-4の確認で、デフォルトのコミット作成者とCredential Helperが適用されていること。

### 5-3. 「追加」の場合

#### ユーザーへの確認項目

| 確認項目 | 用途 | 参照する章 |
|---|---|---|
| 追加するフォルダのパス | `includeIf "gitdir:..."`（末尾は `/`） | 3-2 |
| そのフォルダで使うGitHubアカウント | 既存の設定ファイルを流用するか、3-1で新しく作るか | 3-1, 3-2 |
| （新しいアカウントの場合）コミット作成者名・メールアドレス | 新しい設定ファイルの `[user]` | 3-1 |

アカウントは `gh auth status` の一覧と、既存の設定ファイルが使っているアカウントを添えて選んでもらってください。

#### AIの確認項目

| 確認項目 | 確認方法 | 参照する章 |
|---|---|---|
| 使用するアカウントが `gh` にログイン済みか | `gh auth status` / `gh auth token --hostname github.com --user <アカウント> > /dev/null` | 3-1 |
| 既存の設定ファイルと、それぞれが使うアカウント | `git config --global --get-regexp '^includeif\.'` で `path` を一覧し、各ファイルの `helper` 行の引数を確認 | 3-1 |
| 既存の `includeIf` とパスが重複していないか | `git config --global --get-regexp '^includeif\.'` | 3-2 |
| 追加するフォルダが存在するか | `ls -d <フォルダのパス>` | 3-2 |
| `GH_TOKEN` / `GITHUB_TOKEN` が設定されていないか | `printenv GH_TOKEN >/dev/null && echo set` / `printenv GITHUB_TOKEN >/dev/null && echo set` | 4-2 |

完了条件：3-3の確認で、追加したフォルダ配下の `username=` が期待するアカウント名になっていること。

### 5-4. 共通の注意

- トークンの値を画面やログに出力しないでください。確認には3-3の `sed -n '/^username=/p'` のように `username` 行だけを表示する方法を使ってください。
- `gh auth login` はブラウザ認証を伴うため、ユーザー自身に実行してもらってください。
- `~/.gitconfig` を編集する前に、現在の内容をユーザーに提示し、変更点を説明してから反映してください。
