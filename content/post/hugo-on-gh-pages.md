---
title: "Hugo を GitHub Pages でホストする方法を考える"
date: 2017-11-03T16:25:35+09:00
draft: false
categories: ["web"]
tags: [
  "github",
  "hugo",
  ]
---

今回本ブログを作るにあたって採用した Hugo について、GitHub Pages でホスティングするにあたって検討した内容をまとめてみます。

Hugo は Jekyll と違い GitHub でビルドはしてくれないので、設定ファイルや Markdown と生成結果一式の 2 種類のファイル達をどう GitHub で構成するのかがポイントになります。以下のようなパターンがあります。

| パターン  | 難易度 | ソースの場所 | パス | 独自ドメイン |
| -------- | ------ | ----------- | --- | ----------- |
| &lt;user/org&gt;.github.io | :star::star: | 別のリポジトリ | https://&lt;user/org&gt;.github.io/ | :o: |
| &lt;user/org&gt;.github.io のサブフォルダ | :star: | 同じリポジトリ | https://&lt;user/org&gt;.github.io/&lt;foo&gt;/ | :o: |
| /docs フォルダ | :star: | 同じリポジトリ ※1 | https://&lt;user/org&gt;.github.io/&lt;repo&gt;/ | :o: ※2 |
| gh-pages ブランチ | :star::star: | 同じリポジトリ ※1 | https://&lt;user/org&gt;.github.io/&lt;repo&gt;/ | :o: ※2 |

- ※1: &lt;user/org&gt;.github.io という名前のリポジトリの場合はこのパターンは使えません
- ※2: 独自ドメイン経由でアクセスする際に github.io ではついてしまうお尻の /&lt;repo&gt;/ を省くことができます

パターンの選択にあたって、ソースと公開ファイルを別々で管理したいかどうかは大きなポイントです。同じリポジトリならば、ソースと公開ファイルを一緒にバージョン管理できる利点があり、別のリポジトリ (またはローカル管理) ならばドラフトや設定などを公開せずに運用できる利点があります (GitHub Pages 自体が private リポジトリでは利用できないため)。

今回の私の最大のニーズは mikan.github.io のルートでホストすることでした。従って1番目のパターンしか取りえません。一方、サブディレクトリ (例: mikan.github.io/blog/) を許容する場合は、全てのパターンが選択肢になります。例えば2番目のパターンは、&lt;user/org&gt;.github.io リポジトリの master の直下に Hugo のファイル一式を置いて設定で `publishDir = "blog"` とサブフォルダを指定しつつ、更に index.md 等を置いてルートは Jekyll として利用するというハイブリッドな使い方も可能なパターンになります。

なお、独自ドメイン利用時の注意点として、現在の GitHub Pages では独自ドメイン経由でアクセスする際には HTTPS が使えません (証明書のアップロードとかできるようにして欲しいな・・・)。ただしリバプロはできるので、例えば CloudFlare を使い DNS サーバーを変更して CloudFlare 経由にルーティングすれば、CloudFlare の無料 SSL/TLS 証明書を利用することができます。

様々なセットアップ手順は参考文献の1番目 (公式ドキュメント) に譲りますが、ここでは今回構築した1番目のパターンの手順をご紹介します。

1. GitHub で "blog" (名前は一例) リポジトリを作り、clone できるように README.md か何か置いておく
  - ここに将来のソースファイルが納まる
2. "&lt;user&gt;.github.io" のリポジトリを作り、clone できるように README.md か何か置いておく
  - ここに将来の公開ファイルが納まる
3. `git clone https://github.com<user>/blog.git`
4. Hugo ファイル一式を作り、 `blog` に格納する
  - 別の場所で `hugo new site blog` とかして作ったものを `blog` に突っ込むイメージ
  - `hugo server` しといて http://localhost:1313/ でレンダリング結果を確認しながら執筆 (テーマを指定する場合は `hugo server -t <theme>`、ドラフト表示する場合は `hugo server -D`)
5. いい感じに仕上がったら `Ctrl-C` でサーバーを止めて `rm -rf public` で public を削除
6. `git submodule add -b master git@github.com:<user>/<user>.github.io.git public` で public を公開先リポジトリのサブモジュール化
7. 次のシェルスクリプト (参考文献1より抜粋) を `deploy.sh` として保存、`chmod u+x deploy.sh` して `./deploy.sh`

> ```bash
#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

# Build the project.
hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

# Go To Public folder
cd public
# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master

# Come Back up to the Project Root
cd ..
```

Windows Subsystem for Linux (旧 Bash on Windows) をお使いの方で Hugo 本体が Windows ネイティブの場合は、`hugo` コマンドを `hugo.exe` に置き換えるか、PATH の通っているところで exe へのシンボリックリンクを貼る等で動かせます。

最後に、参考までに私のブログのリポジトリをご紹介します。設定ファイル等参考になれば幸いです。

- ソースリポジトリ: https://github.com/mikan/blog
- 公開ファイルリポジトリ: https://github.com/mikan/mikan.github.io

#### 参考文献

1. [Hugo  | Host on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/)
2. [User, Organization, and Project Pages - User Documentation](https://help.github.com/articles/user-organization-and-project-pages/)
3. [Configuring a publishing source for GitHub Pages - User Documentation](https://help.github.com/articles/configuring-a-publishing-source-for-github-pages/)