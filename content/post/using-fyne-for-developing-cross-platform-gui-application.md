---
title: "Go 製 UI ツールキット Fyne で始めるクロスプラットフォーム GUI アプリケーション開発"
date: 2020-07-12T19:04:00+09:00
draft: false
categories: ["tools"]
tags: [
  "golang",
  "fyne"
]
---

{{<figure src="/img/ss/fyne1.png" class="center">}}

皆さんは **デスクトップアプリ** を作ろう！となったとき、どんな UI ツールキットで作りますか？
OS が決まっているならば SwiftUI, WPF あるいは GTK+ や Qt でしょうか。
色んな OS をサポートしたい場合は、Tcl/Tk, wxWidgets, JavaFX, Xamarin, Electron, React Native なども良いですね。

それでは **Go 言語で** 作りたいとなったときはどうしましょう？
awesome-go<sup>[参考文献1]</sup> によると、以下のような選択肢がありそうです。

| パッケージ | ツールキット/エンジン |
| --- | --- |
| github.com/mattn/go-gtk | GTK+ |
| github.com/gotk3/gotk3 | GTK+ |
| github.com/therecipe/qt | Qt |
| github.com/andlabs/ui | libui |
| github.com/asticode/go-astilectron | Electron |
| github.com/maxence-charriere/go-app | ブラウザ系 |
| github.com/sciter-sdk/go-sciter | ブラウザ系 |
| github.com/dtylman/gowd | ブラウザ系 |
| github.com/wailsapp/wails | ブラウザ系 |
| github.com/zserge/webview | ブラウザ系 |
| fyne.io/fyne | オリジナル |

こうやって分類すると、fyne というのはいったい何者なのだと思いませんか？
そう思ったわたしは実際に試用してみて気に入り、社内で使う簡単な GUI アプリの開発に採用、ほんの一瞬で目的通りのアプリを実際に作れてしまいました。
作ったアプリはのちほど紹介しますが、まずは基本的な使い方と多くの人が遭遇するであろうハマりポイントをいくつか紹介したいと思います。

ただし、現状 Go の GUI 領域はまだまだ発展途上です。Fyne においてもデータバインディングやアニメーション機能が未提供だったりと GUI ツールキットとしては大穴があいているのも事実。限界を承知の上で、過度に期待せずに読んでいただけたらと思います。

### はじめかた

既に Go 言語の開発環境は整っている前提で始めたいと思います。整っていない方は [Getting Started](https://golang.org/doc/install) してください。

では早速コードから。"Hello, world!" を表示するだけの最小構成のプログラムは以下のようになります。

```go
package main

import (
	"fyne.io/fyne/app"
	"fyne.io/fyne/widget"
)

func main() {
	a := app.New()
	w := a.NewWindow("Hello")
	w.SetContent(widget.NewLabel("Hello, world!"))
	w.ShowAndRun()
}
```

デスクトップアプリの本体とは思えない短さですね！

`go.mod` はこんな感じ。

```
module github.com/user/repo

go 1.14

require fyne.io/fyne v1.3.2
```

(バージョンは執筆時点の最新)

さあ、実行してみましょう！

```
go run .
```

{{<figure src="/img/ss/fyne2.png" class="center">}}

表示されましたか？おめでとうございます👏

動きませんでしたか？問題ありません。計算通りです。続きを読んでください。

#### Windows の方

素の Windows で Go のツールチェインだけ入ってるという方、おそらくビルドできないはずです。
Fyne のビルドには C コンパイラが必要です。いくつか選択肢があります。

公式のチュートリアルでは以下が紹介されていました:

- [MSYS2](https://www.msys2.org/)
- [TDM-GCC](https://jmeubank.github.io/tdm-gcc/download/)
- [Cygwin](https://www.cygwin.com/)

私が使っているのは [mingw-w64](http://mingw-w64.org/doku.php/download/mingw-builds) です。MSYS2 にもこれが入っています。
[Chocolatey](https://chocolatey.org/) パッケージマネージャーをお使いの方は `choco install mingw` で `mingw-w64` が一発で入ります！

導入後は cc コマンドにパスが通っていることを確認してください。

#### macOS の方

[Xcode](https://apps.apple.com/us/app/xcode/id497799835?mt=12) とその command line tools が必要です。
command line tools の導入は `xcode-select --install` でいけます。
導入済であっても、macOS をアップグレードした後とかに再度このコマンドを打たないといけないことがあります。

#### Linux の方

gcc に加え、X11 関係の開発用ヘッダファイル等が必要になります。
Linux/Ubuntu では `apt install gcc xorg-dev libgl1-meta-dev`, Fedora とかでは `dnf install gcc libXcursor-devel libXrandr-devel mesa-libGL-devel libXi-devel libXinerama-devel`, Arch では `pacman -S xorg-server-devel` です。

### ライトテーマ

Fyne でアプリを起動して、ダークモードっぽい見た目になるのに驚いた人もいるかもしれません。
Fyne ではダークテーマがデフォルトになっています。ライトテーマもあります。環境変数を設定するだけで切り替わるので、早速試してみましょう！

Unix 系の方は、

```
FYNE_THEME=light go run .
```

Windows (コマンドプロンプト) の方は、

```
set FYNE_THEME=light
go run .
```

PowerShell の方は、

```powershell
$Env:FYNE_THEME='light'
go run .
```

(クロスプラットフォームの手順の説明はたいへんだなぁ)

{{<figure src="/img/ss/fyne3.png" class="center">}}

わずか環境変数1つで驚きの白さに！

### レイアウト

基本的には、複数要素を垂直に並べる `VBox`, 水平に並べる `HBox` を組合せてレイアウトします。
先程のコードの `w.SetContent` 部を以下に書き換えてみましょう。

```go
w.SetContent(widget.NewVBox(
    widget.NewLabel("Label 1"),
    widget.NewLabel("Label 2"),
))
```

{{<figure src="/img/ss/fyne4.png" class="center">}}

縦に2つのラベルが並びました。

次は `HBox`。

```go
w.SetContent(widget.NewHBox(
    widget.NewLabel("Label 1"),
    widget.NewLabel("Label 2"),
))
```

{{<figure src="/img/ss/fyne5.png" class="center">}}

横に2つのラベルが並びました。

入れ子もできます。

```go
w.SetContent(widget.NewVBox(
    widget.NewLabel("Label 1"),
    widget.NewLabel("Label 2"),
    widget.NewHBox(
        widget.NewLabel("Label 3"),
        widget.NewLabel("Label 4"),
    ),
))
```

{{<figure src="/img/ss/fyne6.png" class="center">}}

自由自在！

また、タブやアコーディオン、スクロール対応などのいくつかのコンテナが用意されています。

```go
w.SetContent(widget.NewTabContainer(
    widget.NewTabItem("Tab1", widget.NewLabel("Label 1")),
    widget.NewTabItem("Tab2", widget.NewLabel("Label 2")),
))
```

{{<figure src="/img/ss/fyne7.png" class="center">}}

簡単ですね！

### ウィジェット

UI ツールキットはウィジェットがあればあるほど嬉しいです。しかし・・・ Fyne はあまりありません。
なによりデータバインディングに未対応 (v2 で計画) なので、テーブル系がないのは致命的です。
とはいえ、いまあるものは簡単に使えます。また本記事では割愛しますが拡張もできるようになっています。

ありったけのウィジェットを並べてみました。

```go
w.SetContent(widget.NewVBox(
    widget.NewGroup("Button", widget.NewButton("Button", func() {})),
    widget.NewGroup("Check", widget.NewCheck("Check", func(_ bool) {})),
    widget.NewGroup("Entry", widget.NewEntry()),
    widget.NewGroup("PasswordEntry", widget.NewPasswordEntry()),
    widget.NewGroup("Form", widget.NewForm(widget.NewFormItem("FormItem", widget.NewEntry()))),
    widget.NewGroup("HyperLink", widget.NewHyperlink("HyperLink", &url.URL{})),
    widget.NewGroup("Icon", widget.NewIcon(theme.FyneLogo())),
    widget.NewGroup("Label", widget.NewLabel("Label")),
    widget.NewGroup("ProgressBar", widget.NewProgressBar()),
    widget.NewGroup("Radio", widget.NewRadio([]string{"Option1", "Option2"}, func(_ string) {})),
    widget.NewGroup("Select", widget.NewSelect([]string{"Select"}, func(_ string) {})),
    widget.NewGroup("ToolBar", widget.NewToolbar(widget.NewToolbarAction(theme.DocumentSaveIcon(), func() {}))),
))
```

{{<figure src="/img/ss/fyne8.png" class="center">}}

### 日本語表示

冒頭に張ったスクショには日本語が表示されていましたが、実は Fyne は日本語表示に対応していません！
リポジトリを見ると Noto Sans が同梱されているのが確認できますが、日本語対応の CJK JP ではありません。

ですがもちろん解決策があります。テーマを切り替えたときと同じように環境変数でフォントを指定するだけです！

macOS:

```
FYNE_FONT=/System/Library/Fonts/AquaKana.ttc go run .
```

Windows 10 PowerShell:

```
$Env:FYNE_FONT='C:\Windows\Fonts\YuGothM.ttc'
go run .
```

ヒラギノ角ゴとか指定しようとしたら bad TTF version というエラーで落ちてしまいました。現状では指定できるフォントは限られているようです。

```
2020/07/12 08:34:21 Fyne error:  font load error
2020/07/12 08:34:21   Cause: freetype: invalid TrueType format: bad TTF version
2020/07/12 08:34:21   At: go/pkg/mod/fyne.io/fyne@v1.3.2/internal/painter/font.go:20
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x1d4 pc=0x42a5c31]
```

なお、 `Label` や `HyperLink` ではスタイルを指定できる `NewXxxWithStyle()` という関数もあり、こんな感じで太字・斜体・等幅が指定できます。

```
widget.NewLabelWithStyle("らべる", fyne.TextAlignCenter, fyne.TextStyle{
    Bold: true,
    Italic: true,
    Monospace: true,
})
```

しかしフォントの指定の口は環境変数1つのみです。Fyne のコードを見る限り、ファイル名に `Regular` が含まれていると、その部分を `Bold`, `BoldItalic`, `Italic` に変換するようです。それ以外は指定しても効きません。この実装はちょっと不便です (しかも undocumented)。
また等幅フォントについては `FYNE_FONT` 環境変数ではなく `FYNE_FONT_MONOSPACE` 環境変数で指定するように実装されています。

実際にアプリケーションとしてパッケージングして配布することを考えても、環境変数で指定するというのはあまり都合の良い方式ではありません。
いまのところ現実的な解決策としては、

- `TextStyle` を使いたいときは自分でフォントを (名前を整えた上で) 成果物に同梱し、 `app.New()` より前に環境変数に設定
- `TextStyle` を使わないのであれば、実行 OS から適切なシステムフォントを探し、 `app.New()` より前に環境変数に設定

とするのが良さそうです。

冒頭の例では、以下のようなコードになっています。ファイルが存在するか判定して、あれば環境変数に設定してから `app.New()` します。

```go
fontPath := "/System/Library/Fonts/AquaKana.ttc"
if _, err := os.Stat(fontPath); err == nil {
    os.Setenv("FYNE_FONT", fontPath)
}
a := app.New()
w := a.NewWindow("Hello")
```

### パッケージング

Fyne で作ったデスクトップアプリは、 `go run` で動いたことからもわかるようにそのまま `go build` したものを配布しても構いません。
しかし `go build` だけだと Windows の場合は exe ファイルにアイコンがない上、コマンドプロンプト画面 (いわゆる DOS 窓) が出現してしまいます。そして Mac の場合は当然ながら `.app` 形式のアプリではなく生のバイナリファイルで、デスクトップアプリらしさがありません。Linux もデスクトップ環境が読み込めるパスでアイコンを置いておきたいものです。

というわけでパッケージングコマンドが提供されています。導入方法は簡単。

```
go get fyne.io/fyne/cmd/fyne
```

使い方もシンプルです。適当なアイコンを png ファイルで用意しておいて・・・

```
fyne package -icon icon.png
```

たったこれだけです。`-icon` のほかクロスコンパイル用の `-os <os>` があります。ただし前述のプラットフォーム事前条件を満たせないとクロスコンパイルは成功しません。
あとは `-release` というリリースビルド用オプションもありますが、手元で試したところ `true` でも `false` でも同じバイナリが出力されました・・・。

`fyne package` 以外には `fyne bundle <file|directory>` なんてサブコマンドもあります。
静的コンテンツ (ファイルやディレクトリ) をバイナリに埋め込むことができます。

### おまけ: Windows で exec.Command で DOS 窓が出る

Windows 限定の話ですが、 `fyne package` でバイナリを作るとアプリ起動時にコマンドプロンプト画面 (いわゆる DOS 窓) が出ないよう対策してくれるということを先程紹介しました。
これは何も `fyne` の魔法ではなく、内部で `go build` を以下のように実行しているだけです:

```
go build -ldflags -H=windowsgui .
```

しかしこの副作用なのか、GUI から何らかの外部コマンドを実行するコードを `fyne package` した exe を実行すると、この外部コマンドの DOS 窓が出現するという現象に遭遇しました。

ちょっと長いですが次のようなコードを動かしてみます。

```go
package main

import (
	"os/exec"

	"fyne.io/fyne/app"
	"fyne.io/fyne/widget"
)

func main() {
	a := app.New()
	w := a.NewWindow("ping")
	label := widget.NewLabel("Not executed")
	w.SetContent(widget.NewVBox(
		widget.NewButton("Button", func() {
			label.SetText("Executing...")
			if err := exec.Command("ping", "-n", "3", "1.1.1.1").Run(); err != nil {
				label.SetText("ERROR: " + err.Error())
				return
			}
			label.SetText("Success")
		}),
		label,
	))
	w.ShowAndRun()
}
```

先程の `fyne package` でも `go build -ldflags -H=windowsgui .` でもどちらでも、確かに exe 起動時の DOS 窓は出なくなりますが、 "Button" を押すと DOS 窓が出現してしまいます。

{{<figure src="/img/ss/fyne9.png" class="center" width="80%">}}

色々と調べたところ、これを解決する方法がありました。 `syscall.SysProcAttr` 構造体の `HideWindow` という Windows 専用項目でした<sup>[参考文献3]</sup>。
先程のコードに適用すると、以下のようになります。これで DOS 窓は出なくなりました。

```go
cmd := exec.Command("ping", "-n", "3", "1.1.1.1")
cmd.SysProcAttr = &syscall.SysProcAttr{HideWindow: true}
if err := cmd.Run(); err != nil {
    label.SetText("ERROR: " + err.Error())
    return
}
```

しかしここで新たな問題が発生します。`HideWindow` は Windows 用の Go ツールチェインにしかないので、他の OS ではコンパイルできなくなるのです。
これを解決するにはもはや Go の Build Constraints 機能に頼るしかないでしょう。まず、Windows かそれ以外かで分岐する部分を別のソースファイルにし、そして関数化します。ファイル名は `cmd_windows.go` とします (`_windows` の部分が Build Constraints の規約)。

`cmd_windows.go`:

```
package main

import (
	"os/exec"
	"syscall"
)

func prepareBackgroundCommand(cmd *exec.Cmd) {
	cmd.SysProcAttr = &syscall.SysProcAttr{HideWindow: true}
}
```

Windows 以外の場合は何もしなくて良いですが、関数は必要です。そこでもうひとつ `cmd.go` を作成します。

`cmd.go`:

```
// +build !windows

package main

import "os/exec"

func prepareBackgroundCommand(_ *exec.Cmd) {
	// no-op
}
```

冒頭に書いた `// +build !windows` は `GOOS` が windows じゃない場合はこのソースをビルドしてねという指示です。

準備ができたら、`exec.Cmd` に適用しましょう。

```go
cmd := exec.Command("ping", "-n", "3", "1.1.1.1")
prepareBackgroundCommand(cmd)
if err := cmd.Run(); err != nil {
    label.SetText("ERROR: " + err.Error())
    return
}
```

これで Windows 以外の OS のビルドが壊れなくなります。めでたしめでたし。

### まとめ

Fyne は GUI ツールキットとして基本的な機能を備え、それなりのルックスのデスクトップアプリケーションを簡単に開発できることが分かりました。
しかも、今回は触れませんでしたが Android や iOS で動くアプリも作れるようになっているなど、デスクトップだけでなくモバイルもカバーできる真のクロスプラットフォームなツールキットを目指しているのも心強いです。Fyne の開発は活発に進められており、今後の発展にも大いに期待できます。

一方で、現時点ではまだ機能が出揃っていないという点でフィットしなかったり、あるいは開発環境構築でつまづいたり、新たな問題に遭遇したりして挫折することもありえるツールキットでもあります。問題を解決するには Go や OS の一段深い理解が必要になることもあるでしょう。
万人にお勧めできるツールキットではない点も付け加えないといけません。

なお、本記事を作成する元ネタになったアプリは GitHub で公開しています。
ARPG (あーぷじー) という名前で、LAN 内で IP アドレスと MAC アドレスを相互に解決できる GUI アプリです。中身は単に OS のコマンドを叩いているだけですが、どうしても業務で GUI が欲しかったのでサクッと作りました。コードも短いですが、本記事で紹介したノウハウが詰まっています。

{{<figure src="/img/ss/arpg1.png" class="center" link="https://github.com/mikan/arpg">}}

> mikan/arpg: IP address / MAC address resolving GUI tool
>
> https://github.com/mikan/arpg

何かの参考になれば幸いです。

Happy hacking!

#### 参考文献

1. [avelino/awesome-go: A curated list of awesome Go frameworks, libraries and software](https://github.com/avelino/awesome-go)
2. [fyne/theme/font at master · fyne-io/fyne](https://github.com/fyne-io/fyne/tree/master/theme/font)
3. [Build golang app (reverse shell ) to run in Windows without it pops up the console (cmd.exe). Any idea, or clues ?? : golang](https://www.reddit.com/r/golang/comments/2c1g3x/build_golang_app_reverse_shell_to_run_in_windows/)
4. [build - The Go Programming Language](https://golang.org/pkg/go/build/#hdr-Build_Constraints)
