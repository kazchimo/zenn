---
title: "ZIO Testの使い方"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zio, scala, test]
published: false
---

## TestAspectの使い方
- zio testではテストのメタデータをTestAspectを使用して設定する
    - テストのbefore/after挙動やテストを走らせる環境など
    
### テストの前処理・後処理をしたい
- `after`/`before`/`around`などのaspectを付与すればいける

### テストを順番に走らせたい
- zio testはデフォルトで各testが非同期に走るっぽい
- dbを使用したテストなどの外部に依存するときには非同期に走らせると状態が競合してうまくテストが通らない
- `sequential`aspectを付与してsuiteを走らせればそのsuiteに含まれるtestは順に同期的に走るようになる
