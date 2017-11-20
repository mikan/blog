---
title: "Go の dep やら govendor やらを GitLab CI で回す"
date: 2017-11-20T22:46:31+09:00
draft: false
categories: ["tools"]
tags: [
  "golang",
  "gitlab",
  "ci/cd",
  ]
---

今回は Go 言語 (Golang) の CI 環境に関する備忘録。難易度は低いのですが、カンペなしですらすら書けないのでソースをぺったんするだけの記事です。

現在職場で使っているリポジトリホスティングサービスはGitLab.com で、皆さんご存知の通り大変素晴らしいツールです。無料なのにあんなことやこんなこともでき、そしてリポジトリブラウザとうまく統合された CI サービスもついてきます。自前で運用する場合は難儀な代物ですが (~~マジで辛い過去が...~~)、本家のマネージドサービスなら安心です。もっとも、過去には DB ぶっ飛び事件などのお騒がせもありましたが:innocent:

{{< figure src="/img/logo/gitlab.png" title="GitLab.com" class="center" link="https://about.gitlab.com/" width="20%" >}}

そんなわけで `.gitlab-ci.yml` の中身をぺったんしていきます。まずは普通に `go build`, `go test` する場合から。

### go build

```yml
image: golang:1.9

variables:
  REPO_NAME: gitlab.com/<YOUR_GROUP>/<YOUR_REPO>

stages:
  - build
  - test

build-project:
  stage: build
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go build ./...

test-project:
  stage: test
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go test ./... -cover
  coverage: '/coverage: \d+\.\d+% of statements/'
```

`<YOUR_GROUP>/<YOUR_REPO>` はあなたのリポジトリ通り書き換えてください。なおこの例では build と test でステージを分けていますが、ひとまとめのが断然速いです！とはいえこれでステージの切り方がわかりますし、Pipeline as Code なお気持ちになれます。

冒頭の `image: golang:1.9` は、そのまんま Docker Hub のイメージです。つまりこれ:

> https://hub.docker.com/_/golang/

適切なイメージ (バージョン) を選びましょう。

`test` ステージにある最後の `coverage` にある文字列は、コンソールログからカバレッジのパーセンテージを拾う正規表現です。これを書いておくと、ジョブのログに数値が出ます。

{{< figure src="/img/ss/gitlab-ci1.png" title="CI / CD -> Jobs" class="center" >}}

・・・そんだけです。グラフとかはない模様。設定が超絶シンプルかつスーパー汎用的な反面、さっと結果のみを確認する以外のこと (分析とか) はなにもできないのはトレードオフといったところか。なお、ログに複数出現すると最後のもの**だけ**がここに出るため、複数パッケージ (まさに `./...` でやっている) の場合はマージされません。そりゃそうだよね。ここは割り切るか、めっちゃ頑張るかのどちらかです。私は前者です😄

### dep

お次は golang.org 公式の依存性管理ツールとなった [dep](https://github.com/golang/dep) です。go get して取ってきていますが、いずれ go コマンドに統合された暁にはそれも不要となることでしょう。

```yml
image: golang:1.9

variables:
  REPO_NAME: gitlab.com/<YOUR_GROUP>/<YOUR_REPO>

stages:
  - build
  - test

build-project:
  stage: build
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go get github.com/golang/dep/cmd/dep
    - $GOPATH/bin/dep ensure
    - go build ./...

test-project:
  stage: test
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go get github.com/golang/dep/cmd/dep
    - $GOPATH/bin/dep ensure
    - go test ./... -cover
  coverage: '/coverage: \d+\.\d+% of statements/'
```

いつものように魔法のおまじない `dep ensure` を詠唱すると、英霊たち (/vendor 以下) を召喚できます。

### govendor

最後は `govendor` です。私実は dep よりこっちのが好きなんです。dep は凝った toml フォーマットなんて採用しちゃってますが、こちらは純粋無垢な json ファイルが 1 個ぺろっと自動生成されるだけのシンプル設計。dep 同様 Heroku デプロイにも使えます。素晴らしい・・・✨

```yml
image: golang:1.9

variables:
  REPO_NAME: gitlab.com/<YOUR_GROUP>/<YOUR_REPO>

stages:
  - build
  - test

build-project:
  stage: build
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go get github.com/kardianos/govendor
    - $GOPATH/bin/govendor sync
    - $GOPATH/bin/govendor build ./...

test-project:
  stage: test
  script:
    - mkdir -p $GOPATH/src/$REPO_NAME
    - mv $CI_PROJECT_DIR/* $GOPATH/src/$REPO_NAME
    - cd $GOPATH/src/$REPO_NAME
    - go get github.com/kardianos/govendor
    - $GOPATH/bin/govendor sync
    - $GOPATH/bin/govendor test ./... -cover
  coverage: '/coverage: \d+\.\d+% of statements/'
```

govendor コマンドが持つ go コマンドへのパススルーもちゃっかり使ってみました。簡単でしたね。

#### 参考文献

1. [How to create a CI/CD pipeline with Auto Deploy to Kubernetes using GitLab and Helm | GitLab](https://about.gitlab.com/2017/09/21/how-to-create-ci-cd-pipeline-with-autodeploy-to-kubernetes-using-gitlab-and-helm/)
2. [Heroku Go Support | Heroku Dev Center](https://devcenter.heroku.com/articles/go-support)