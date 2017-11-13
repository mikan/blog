---
title: "GitHub OAuth の org スコープの適用と運用"
date: 2017-11-13T20:34:21+09:00
draft: false
categories: ["web"]
tags: [
  "github",
  "oauth",
  "java",
  "spring-boot",
  "spring-security",
  "vaadin-framework",
  ]
---

GitHub の OAuth をするとき、様々な追加権限の要求を突きつけられた経験はありますか？ありますよね？例えばこんな画面。

{{<figure src="/img/ss/github-oauth1.png" class="center">}}

今回はそんな認可画面遷移の一瞬にまつわるメモです。

OAuth には scope という概念があり<sup>[参考文献3]</sup>、GitHub では以下のような scope が定義されています<sup>[参考文献1]</sup>。

- `repo` (`repo:status`, `repo_deployment`, `public_repo`, `repo:invite`)
- `admin:org` (`write:org`, `read:org`)
- `admin:publickey` (`write:public_key`, `read:public_key`)
- `admin:repo_hook` (`write:repo_hook`, `read:repo_hook`)
- `admin:org_hook`
- `gist`
- `notifications`
- `user` (`read:user`, `user:email`, `user:follow`, `delete_repo`)
- `admin:gpg_key` (`write:gpg_key`, `read:gpg_key`)

カッコはスコープの包含関係を示しています。例えば、`user:email` はユーザープロフィールのうち Email 情報だけ取得でき、`user` スコープは `read:user` と `user:email` と `user:follow` と `delete_repo` の各スコープが可能な権限を足し合わせたスコープです。なお、スコープはスペース区切り (URL エンコードでは `%20`) で一度に複数指定することもできます。
要求したスコープは、ユーザーの承諾なしに認可されることはなく、また認可トークンは各 OAuth アプリに閉じています。
この仕組みを活用することで、悪い OAuth アプリからの攻撃や認可情報の流出による被害を最小限に留められ、そして問題を発見した際には対象を無効化するだけで対処することができるわけです。

今回は、3年前に立ち上げ現在も継続しているオンライン読書会「[AOSN読書会](https://aosn.ws)」の課題本投票システム「[Mosaic](https://vote.aosn.ws)」がお題です。

{{<figure src="/img/ss/mosaic1.png" class="center" width="80%">}}

このシステムは GitHub の [AOSN Organization](https://github.com/aosn) のメンバーのみに投票権を付与させるため、`read:org` スコープの認可を要求します (冒頭の画像がその認可画面です)。なお、実装は Java で Spring Boot, Spring Security と Vaadin Framework 8 を用いており、IBM Bluemix と [VULTR VPS 東京リージョン](https://www.vultr.com/?ref=7053029)の2拠点で運用しています。

ユーザーが特定の Organization に属しているか否かという情報は、ユーザーが公開・非公開を選べる類の情報であり、デフォルトは非公開です。さらにOwner はメンバーの公開状態を Public から Private にはできても、逆はできません。

{{<figure src="/img/ss/github-org1.png" class="center" title="Organization -> People">}}

このような性質の情報を取り扱うことから、ユーザーの所属組織一覧を取得する API `GET /user/orgs` では `read:org` または `user` のどちらかのスコープの認可がないとアクセスすらできません<sup>[参考文献2]</sup>。投票システムのユースケースでは、`user` は大きすぎるスコープであるため、`read:org` を要求しています。どうしても要求したくないという場合は、誰か一人のユーザーに対象の Organization のメンバー一覧を取得させ、それをその他のユーザーと使いまわすといった設計も可能ですが、誰か一人のユーザーのクレデンシャルをシステムに生贄として捧げる必要がある事に加え、フレームワークの支援を得るのも難しくなり、投票システムのユースケースではオーバーエンジニアリングでしょう。

なお、本題からは外れますが spring-security-oauth2 で scope を指定する方法はとても簡単です。`application.yml` に `scope` を書くだけです。<sup>[参考文献4]</sup>。`ConfigurationProperties` を使った場合の例は次の通り。

```java
    @Bean
    @ConfigurationProperties("github")
    public ClientResources github() {
        return new ClientResources();
    }
```

上記の場合の `application.yml`:

```yml
github:
  client:
    clientId: XXXXXXXX
    clientSecret: XXXXXXXX
    accessTokenUri: https://github.com/login/oauth/access_token
    userAuthorizationUri: https://github.com/login/oauth/authorize
    clientAuthenticationScheme: form
    scope: read:org
  resource:
    userInfoUri: https://api.github.com/user
```

さて、OAuth アプリが無事 `read:org` を得られるようになったところで、冒頭の認可画面をふりかえってみます。

{{<figure src="/img/ss/github-oauth1.png" class="center">}}

Organization ごとに表示されている :heavy_check_mark: や :x: は一体何なのでしょう？このインジケータは、皆さんの期待通り :heavy_check_mark: の Organization の `read:org` はこのアプリに認可し、:x: の Organization の `read:org` はこのアプリに認可しないことを示しています。所属している Organization が 4 つでも、認可されている Organization が 2 つならば、`GET /user/orgs` で帰ってくる Organization も認可された 2 つのみとなります (手元で検証済です！)。ではデフォルトで :heavy_check_mark: が入っている (そして外すことができない！) Organization と、デフォルトで :x: で、手動で Grant できる Organization とで一体何が違うのでしょうか？

{{<figure src="/img/ss/github-org2.png" class="center" width="80%" title="Organization -> Settings -> Third-party access (No restrictions)">}}

答えは Organization ごとの **Third-party access** 設定です<sup>[参考文献5]</sup>。これのポリシーが「No restrictions」(無制限) となっている Organization が、認可画面の拒否できない :heavy_check_mark: になります。一方「Access Restricted」(アクセス制限済) となっている Organization は、認可画面でデフォルトで :x: となります。

{{<figure src="/img/ss/github-org3.png" class="center" width="80%" title="Organization -> Settings -> Third-party access (Access restricted)">}}

No restrictions 状態の Organization には大きなリスクがあります。なぜならメンバーの **誰か1人が** 悪意ある OAuth アプリを認可したり、あるいは認可情報が流出して悪用されてしまえば、No restrictions の Organization は無条件でアクセス許可されるため内部情報が丸裸となってしまい、さらに `write` や `admin` まで認可してしまった場合は、改ざんや消失のおそれまであるからです。Access Restricted な Organization ならば、OAuth アプリを認可しても、そのときに Grant ボタンさえ押さなければ設定した Organization への被害を食い止められます (特に、何も考えずに先に進もうとする初心者は端っこにある小さな Grant ボタンより先に緑の Authorize ボタンを押すはずです)。従って、 **特段の理由がない限り全ての GitHub Organization はまっさきに Access Restricted にすべき** といえます。

`read:org` や `admin:org` のアプリなんてめったにない？いえいえ、`org` スコープはあくまで一つの例で、CI/CD サービスなどに多い `repo` 系を要求してくるアプリも同様です。認可時に :heavy_check_mark: になっている Organization や、手動で Grant した Organization は、対象 OAuth アプリにそれらに属する全てのリポジトリの認可を与えることになります (public / private 別の scope 分類はあり)。これらの認可を下す際は、しっかりレビューをしてください。そしてもしあなたの Organization が No restrictions のままだったら、すぐさま管理者に Third-party access restriction の有効化を依頼してください。

{{<figure src="/img/ss/github-org4.png" class="center" width="80%" title="Organization -> Settings -> Third-party access (Approved apps)">}}

有効化した後は、上記のページのように手動で Grant した OAuth Apps 達を定期的にレビュー・棚卸しをしたほうが良いでしょう。不要になったサービスは疑わしいものも含めて積極的に Deny (承認から外) し、常に本当に必要な OAuth アプリにのみ権限を与えておく運用が理想です。あるいは、必要になったらまた認可すればいいわけですから、定期的に全部 Deny してしまう、というアプローチもありですね。

おまけ: どの Organization にも属していないユーザーが Organization や Repository 系の OAuth アプリを認可しようとすると、Organization に関する追加権限は表示されません。

{{<figure src="/img/ss/github-oauth2.png" class="center">}}

操作マニュアル職人はいろんなパターンがあって骨が折れますね :innocent:

#### 参考文献

1. [About scopes for OAuth Apps | GitHub Developer Guide](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-scopes-for-oauth-apps/)
2. [Organizations | GitHub Developer Guide](https://developer.github.com/v3/orgs/#list-your-organizations)
3. [draft-ietf-oauth-v2-22 - The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-3.3)
4. [Tutorial · Spring Boot and OAuth2](https://spring.io/guides/tutorials/spring-boot-oauth2/)
5. [About OAuth App access restrictions - User Documentation](https://help.github.com/articles/about-oauth-app-access-restrictions/)