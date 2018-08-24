---
title: "Jackson と Kotlin の組み合わせで起きる不思議な現象の解析"
date: 2018-08-24T14:00:00+09:00
draft: false
categories: ["web"]
tags: [
  "kotlin",
  "gradle",
  "java",
  "book",
  ]
---

Jackson といえば、Java 界隈ではとても有名な JSON ライブラリですが、 Kotlin (for JVM) でもそのパワーを存分に発揮することができます。特にクラスと JSON をマッピングする jackson-databind ライブラリは強力です。
なのですが、Kotlin との組み合わせしか起こりえないであろう面白い？現象に遭遇したので、原因と解析アプローチなど少しメモしておこうと思います。

### おさらい

まずは基本的な使い方のおさらい。適当なクラス `Person` を用意して文字列の JSON をマッピングしてみます。

```
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.databind.ObjectMapper

class Person {
    var id = 0L

    @JsonProperty("full-name")
    var fullName = ""

    var married = false
}

val json = """
        {
          "id": 100,
          "full-name": "Yutaka Kato",
          "married": false
        }
    """.trimIndent()

fun main(args: Array<String>) {
    val person = ObjectMapper().readValue<Person>(json, Person::class.java)
    println(person.fullName)
}
```

:bulb: `@JsonProperty` はメンバー名と JSON のキー名が異なる場合に任意の対応付けを指定できるオプションのアノテーションです。

とっても簡単ですね。

と、ここまでは Java と全く同じ使い方ですので、 Gradle に書く依存性はたったこれだけです。

```groovy
dependencies {
    compile 'com.fasterxml.jackson.core:jackson-databind:2.9.6'
}
```

しかしこれでは Kotlin 固有のデータクラス、多様なコンストラクタ、ビルトインクラス等のかゆいところにまでは手が届きません。そこで Kotlin 用の支援モジュールが提供されています<sup>[参考文献2]</sup>。これを利用する場合は次のようになります。

```groovy
dependencies {
    compile 'com.fasterxml.jackson.module:jackson-module-kotlin:2.9.6'
}
```

冒頭の例のデータクラス + jackson-module-kotlin 版 (ついでに immutable 化):

```
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

data class Person(
    val id: Long,

    @JsonProperty("full-name")
    val fullName: String,

    val married: Boolean
)

// val json は省略

fun main(args: Array<String>) {
    val person = jacksonObjectMapper().readValue<Person>(json, Person::class.java)
    println(person)
    // 出力:
    // Person(id=100, fullName=Yutaka Kato, married=false)
}
```

:bulb: `jacksonObjectMapper()` の代わりに `ObjectMapper().registerKotlinModule()` と書くこともできます。

### Kotlin プロパティ名の変更

さて、ここで仕様変更、 `married` を `IsMarried` に変えて JSON 出力してほしいと言われたとします (そんなんあるか！って思うかもしれませんが、半分実話です・・・)。
ひとまず最初の (データクラスでない) `Person` クラスを改修して対応することにしましょう。

```
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.databind.ObjectMapper

class Person {
    var id = 0L

    var fullName = ""

    @JsonProperty("IsMarried")
    var isMarried = false
}

fun main(args: Array<String>) {
    val person = Person()
    person.id = 100
    person.fullName = "Yutaka Kato"
    person.isMarried = false
    val json = ObjectMapper().writeValueAsString(person)
    println(json)
}
```

果たして結果は・・・じゃかじゃん！

```json
{
  "id": 100,
  "fullName": "Yutaka Kato",
  "married": false,
  "IsMarried": false
}
```

:bulb: jq で整形しています。

**変えたはずの married が残っている・・・!?**

おかしいですね。念の為 jackson-module-kotlin のほうでも試してみましょう。

```
import com.fasterxml.jackson.annotation.JsonProperty
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

data class Person(
    val id: Long,

    val fullName: String,

    @JsonProperty("IsMarried")
    val isMarried: Boolean
)

fun main(args: Array<String>) {
    val person = Person(100, "Yutaka Kato", false)
    val json = jacksonObjectMapper().writeValueAsString(person)
    println(json)
}
```

結果。

```json
{
  "id": 100,
  "fullName": "Yutaka Kato",
  "married": false
}
```

**こっちはこっちで変更が反映されていない・・・!?**

大変不思議な現象が発生しました。

### 種明かし

Jackson がマッピングを探す解析対象とするのはクラスのメンバー、つまりフィールドとアクセサです。
一方、Kotlin でクラスのブロックの中に書いた変数・定数は、一見 Java のフィールドのように見えますが、実態は C# などと同じプロパティとなっています。つまり、バッキングフィールドとアクセサを組み合わせた概念です<sup>[参考文献3]</sup>。

これを確かめるために、実際に Kotlin コンパイラがどのような振る舞いをしているのかを調べてみましょう。
まずは改修前の `Person` クラスをコンパイルした結果である Person.class を `javap` コマンドで解析してみます。

```
$ javap -p Person.class 
Compiled from "Person.kt"
public final class Person {
  private long id;
  private java.lang.String fullName;
  private boolean married;
  public final long getId();
  public final void setId(long);
  public final java.lang.String getFullName();
  public final void setFullName(java.lang.String);
  public final boolean getMarried();
  public final void setMarried(boolean);
  public Person();
}
```

各フィールド (`id`, `fullName`, `married`) に対してアクセサ (`getXXX()`, `setXXX()`) が生成されているのがわかります。
その結果、Jackson はこれら 3 つの JSON キーをマッピングすることができます。

改修後の `Person` クラスを見てみると・・・

```
$ javap -p Person2.class 
Compiled from "Person.kt"
public final class Person {
  private long id;
  private java.lang.String fullName;
  private boolean isMarried;
  public final long getId();
  public final void setId(long);
  public final java.lang.String getFullName();
  public final void setFullName(java.lang.String);
  public final boolean isMarried();
  public final void setMarried(boolean);
  public Person();
}
```

フィールド `isMarried` に対するアクセサは `isMarried()` と `setMarried(boolean)` の 2 つになりました。
先程の Jackson の出力では `IsMarried` (フィールド `isMarried` に付けたアノテーションから) に加えて `married` が出現していました。
一体この `married` はなぜ出てきてしまったのでしょうか。

実はこれ、Jackson Databind のごく一般的な仕様です。Jackson Databind は Getter / Setter からプロパティを推測する機能が備わっており、デフォルトで有効になっています<sup>[参考文献4]</sup>。したがって、 `isMarried` フィールドはアノテーションで示した通り `IsMarried` で出力しつつ、 `isMarried()` または `setMarried(boolean)` というアクセサから `married` プロパティを自動検出してしまったのです。

:warning: これはほぼ JavaBeans 標準の命名規約のことを指しているように思われますが、それを示す `USE_STD_BEAN_NAMING` は現在デフォルトで `false` になっており、厳密にはそれとも異なる仕様で動作しています。

試しに `Boolean` を `String` に変えてみます。

```
class Person {
    var id = 0L

    var fullName = ""

    @JsonProperty("IsMarried")
    var isMarried = "false"
}
```

すると、余計なキーは消えました。

```json
{
  "id": 100,
  "fullName": "Yutaka Kato",
  "IsMarried": "false"
}
```

`javap` を見ると・・・

```
$ javap -p Person.class 
Compiled from "Person.kt"
public final class Person {
  private long id;
  private java.lang.String fullName;
  private java.lang.String isMarried;
  public final long getId();
  public final void setId(long);
  public final java.lang.String getFullName();
  public final void setFullName(java.lang.String);
  public final java.lang.String isMarried();
  public final void setMarried(java.lang.String);
  public Person();
}
```

アクセサは `String` の `isMarried()` と `setMarried(String)` になっています。is を用いながらも `Boolean` ではないので、推測されなかったということです。

もう一つ、データクラス + jackson-module-kotlin のほうでなぜ本来意図したフィールドのほうが消えてしまうのかという問題もありました。
クラスファイル上にもアノテーションは残っており、Kotlin の問題ではなさそうです。フィールドとアクセサに同じ名前 (`isMarried`) が出現することで、たまたま (アノテーションのついていない) アクセサのルールだけ後勝ちで拾われている、という推測もできますが、定かではありません。

### まとめ

プロパティ名に get/set/is のようなアクセサにつけるような名前を付けてしまうと、Java Bean 命名規則で動作するフレームワークやライブラリが混乱してしまうという、まあ当たり前ともいえる教訓が得られました。今回示したような例では、いくら仕様が変わってもプログラミング言語上はあくまでプログラミング言語として慣例に従った命名規則をし、正しいマッピングミスマッチ解決手段である `@JsonProperty` を利用するのが正解といえるでしょう。

また、Kotlin や Jackson といった手間を省く便利な技術の裏には必ず自動検出やシンタックスシュガーのような暗黙的な変換が隠されているものです。そうしたベースの技術を知っていることが、障害解析には欠かせないというのもまた教訓かなと思います。
今回の場合は実際の開発業務でパートナーさんが遭遇した実例 (ただし紹介したコードはダミー) なのですが、依頼を受けてすぐに根本原因に思い当たれたのは「[Kotlin イン・アクション](https://amzn.to/2Lm3fO7)」を一通り読んでいたおかげかもしれません。
書籍や公式リファレンスによる体系的な知識は開発をしていく上でやはり必要不可欠ですね。

{{<figure src="/img/cover/kia_ja.jpg" class="center" alt="Kotlin イン・アクション" width="20%" link="https://amzn.to/2Lm3fO7">}}

・・・ところで私の `married` プロパティはいつになったら `true` になるのでしょうか。

Happy Hacking!

#### 参考文献

1. [FasterXML/jackson: Main Portal page for the Jackson project](https://github.com/FasterXML/jackson)
2. [FasterXML/jackson-module-kotlin: Module that adds support for serialization/deserialization of Kotlin (http://kotlinlang.org) classes and data classes.](https://github.com/FasterXML/jackson-module-kotlin)
3. [Kotlin イン・アクション](https://amzn.to/2Lm3fO7) 2.2.1 プロパティ (P30)
4. [Mapper Features · FasterXML/jackson-databind Wiki](https://github.com/FasterXML/jackson-databind/wiki/Mapper-Features)
