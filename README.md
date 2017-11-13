mikan.aosn.ws
=============

[mikan](https://github.com/mikan) の技術ブログの中身です。[Hugo](https://gohugo.io/) と GitHub Pages を利用しています。

### 基本操作

新しい記事を作る:

```
hugo new post/hello-world.md
vi content/post/hello-world.md
```

ドラフトの表示をチェックする:

```
hugo server -D
```

公開前チェックリスト:

- [ ] 参考文献を十分なだけ載せた
- [ ] 誤字・脱字がないことを確認した
- [ ] アンカーや画像のリンク切れがないことを確認した
- [ ] `draft: false` にした
- [ ] 適切な `date:` に修正した
- [ ] 適切なカテゴリを設定した
- [ ] 適切なタグを設定した

以下のスクリプトで公開する:

```
./deploy.sh
```
