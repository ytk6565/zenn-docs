---
title: "ディレクトリベースで GitHub アカウントを切り替える"
emoji: "🤹"
type: "tech"
topics: ["github", "git"]
published: true
---

# はじめに

初めて Zenn に投稿します。宜しくお願いいたします。

会社用と個人用など複数の GitHub アカウントを利用中に

- `~/.ssh/config` に独自のホストを指定して SSH キーを切り替えているがリポジトリを設定する際にホストを変更するのが面倒。
- 誤ったアカウントでコミットしてしまった。
- 自身のコミットを証明できない。

などの問題に直面したため、解決方法をまとめることにしました。

## 前提

- macOS を利用しております。他の OS をご利用の場合は適宜お読み替えください。
- GitHub を利用している方を想定して書いております。
- git のバージョンは `2.30.0` を利用しております。
  - [core.sshCommand](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand) を利用するには `2.10.0` 以上のバージョンをご利用ください。
  - [Includes](https://git-scm.com/docs/git-config#_includes) を利用するには `2.13.0` 以上のバージョンをご利用ください。
- 今回は下記のディレクトリ構成でアカウントの切替を行います。
  - `~/git/business/` 以下を会社用アカウントで利用するリポジトリ。
  - `~/git/personal/` 以下を個人用アカウントで利用するリポジトリ。

# SSH

## SSH キーを生成する

```sh
$ cd ~/.ssh
$ ssh-keygen -t ed25519 -C "business@example.com" -f id_ed25519_github_business
$ ssh-keygen -t ed25519 -C "personal@example.com" -f id_ed25519_github_personal
```

## GitHub アカウントに SSH キーを追加する

それぞれの [GitHub アカウントへの新しい SSH キーの追加](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account) を行う。

```sh
# copy business public key
$ pbcopy < ~/.ssh/id_ed25519_github_business.pub
# copy personal public key
$ pbcopy < ~/.ssh/id_ed25519_github_personal.pub
```

# GPG

コミットに署名するため GPG の対応を行います。
[Git コミットで作成者を偽装する方法／署名付きコミットでの対策](https://qiita.com/Siketyan/items/bb869f740a53a3bf169e) という記事が大変参考になりました。

## GPG キーを生成する

[GPG キーの生成](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/generating-a-new-gpg-key#gpg-%E3%82%AD%E3%83%BC%E3%81%AE%E7%94%9F%E6%88%90) を参考に GPG キーを生成する。

**※メールアドレスはそれぞれのアカウントで利用しているメールアドレスを指定する。**

```sh
# gpg をインストール
$ brew install gpg
$ gpg --version
gpg (GnuPG) 2.2.26
...

# business
$ gpg --full-generate-key
# personal
$ gpg --full-generate-key
```

## GPG キーを確認する

```sh
$ gpg --list-secret-keys --keyid-format LONG
/Users/username/.gnupg/pubring.kbx
--------------------------------
sec   rsa4096/BUSINESS_1234567 2021-01-01 [SC]
      ****************************************
uid                 [ultimate] Taro Yamada (Business Github key) <business@example.com>
ssb   rsa4096/**************** 2021-01-01 [E]

sec   rsa4096/PERSONAL_1234567 2021-01-01 [SC]
      ****************************************
uid                 [ultimate] Taro Yamada (Personal Github key) <personal@example.com>
ssb   rsa4096/**************** 2021-01-01 [E]
```

## GitHub アカウントに GPG キーの追加する

下記コマンドで GPG キーをコピーし [GitHub アカウントへの新しい GPG キーの追加](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/adding-a-new-gpg-key-to-your-github-account) を行う。

```sh
# business
$ gpg --armor --export BUSINESS_1234567
# personal
$ gpg --armor --export PERSONAL_1234567
```

### pinentry-mac をインストール

GPG キーの生成時に設定したパスフレーズをコミットの度に入力するのは面倒なので `pinentry-mac` をインストールしてパスフレーズをキーチェーンに保存し入力を初回のみで済ませる。

```sh
$ brew install pinentry-mac
```

`~/.gnupg/gpg-agent.conf` に以下を追記する。

```conf
# Intel
pinentry-program /usr/local/bin/pinentry-mac
# Apple Silicon
pinentry-program /opt/homebrew/bin/pinentry-mac
```

# ディレクトリによってアカウントを切り替える

## アカウントごとに設定ファイルを作成する

ベースとなる設定ファイルは `~/.gitconfig` ですが、
加えて `~/.gitconfig.business` と `~/.gitconfig.personal` を作成しそれぞれのアカウントに必要な設定を適用します。

```sh
$ cd
$ touch .gitconfig.business
$ touch .gitconfig.personal
```

### アカウントごとに適切な値を設定する

- `core.sshCommand` に SSH 接続時のコマンドの `-i` オプションで、利用する **SSH キー**を指定する。
- `user.signingkey` に **GPG キー**を指定する。

※ `gpg.program` と `commit.gpgSign` に関してはグローバルに設定しても良いですが GitHub 以外のサービスを利用することを考慮してあえて個別に設定しております。

```properties:.gitconfig.business
[core]
  sshCommand = "ssh -i ~/.ssh/id_ed25519_github_business -F /dev/null"
[user]
  name = user_business
  email = business@example.com
  signingkey = BUSINESS_1234567
[gpg]
  program = gpg
[commit]
  gpgSign = true
```

```properties:.gitconfig.personal
[core]
  sshCommand = "ssh -i ~/.ssh/id_ed25519_github_personal -F /dev/null"
[user]
  name = user_personal
  email = personal@example.com
  signingkey = PERSONAL_1234567
[gpg]
  program = gpg
[commit]
  gpgSign = true
```

## ディレクトリによってアカウントを切り替える

[Includes](https://git-scm.com/docs/git-config#_includes) の `includeIf` を利用し、指定したディレクトリ配下で利用する設定ファイルを切り替える。

```sh
$ git config --global includeIf."gitdir:~/git/business/".path ".gitconfig.business"
$ git config --global includeIf."gitdir:~/git/personal/".path ".gitconfig.personal"
```

### アカウントの切り替えを確認する

- コミットする際に `pinentry-mac` ウインドウが表示されるため `Save in Keychain` にチェックを入れてパスフレーズを入力する。
- push 後に GitHub で該当のコミットを確認し **Verified** マークが表示されていれば GPG キーによってコミットに署名がされていることが確認できます。

```sh:business
$ cd ~/git/business
$ git clone git@github.com:user_business/business-repository.git
Cloning into 'business-repository'...
...
Resolving deltas: 100% (999/999), done # or "warning: You appear to have cloned an empty repository."

$ cd ./business-repository
$ git config core.sshCommand # output: ssh -i ~/.ssh/id_ed25519_github_business -F /dev/null
$ git config user.name # output: user_business
$ git config user.email # output: business@example.com
$ git config user.signingKey # output: BUSINESS_1234567
$ git commit --allow-empty -m "Initial commit" # Enter the passphrase in the window
$ git push
```

```sh:personal
$ cd ~/git/personal
$ git clone git@github.com:user_personal/personal-repository.git
Cloning into 'personal-repository'...
...
Resolving deltas: 100% (999/999), done # or "warning: You appear to have cloned an empty repository."

$ cd ./personal-repository
$ git config core.sshCommand # output: ssh -i ~/.ssh/id_ed25519_github_personal -F /dev/null
$ git config user.name # output: user_personal
$ git config user.email # output: personal@example.com
$ git config user.signingKey # output: PERSONAL_1234567
$ git commit --allow-empty -m "Initial commit" # Enter the passphrase in the window
$ git push
```

ついでに、デフォルトブランチが `main` でない場合はお好みで対応。

```sh
$ git config --global init.defaultBranch # output: master
$ git config --global init.defaultBranch main
$ git config --global init.defaultBranch # output: main
```

# おわりに

複数の GitHub アカウントの利用が安全に行えるようになりました。
[必須コミット署名を有効にする](https://docs.github.com/ja/free-pro-team@latest/github/administering-a-repository/enabling-required-commit-signing) などを行い特定のブランチの保護も対応するとなお良いかと思います。

最後までお読みいただきありがとうございます。
記事に対するリアクションや情報の誤り、誤字脱字などございましたらコメントや[記事を管理しているリポジトリ](https://github.com/ytk6565/zenn-docs)までお願いいたします。

# 参考にさせていただいたページ

- [Git コミットで作成者を偽装する方法／署名付きコミットでの対策](https://qiita.com/Siketyan/items/bb869f740a53a3bf169e)
- [ディレクトリによって git アカウントを自動で切り替える設定(includeIf)](https://qiita.com/shungo_m/items/387d70b1645ae03435b5)
- [git clone 時に秘密鍵を指定する](https://qiita.com/sonots/items/826b90b085f294f93acf)
