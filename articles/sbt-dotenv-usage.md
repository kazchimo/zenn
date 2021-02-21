---
title: ""
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## Test環境での環境変数の設定方法
`Test / envFileName`と`fork in Test`を設定する必要がある。
以下の例では`test.env`というファイルの中身がテスト環境での環境変数として読まれる。

```scala
// build.sbt
fork in Test := true
Test / envFileName := "test.env"
```
