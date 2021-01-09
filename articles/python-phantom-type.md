---
title: "PythonでPhantom Type(幽霊型)を使って静的にプログラムの欠陥を発見する"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Python, 型]
published: true
---

## はじめに
HaskellやScalaなどの静的型付け言語で用いられるテクニックとしてphantom typeというものがある。
Pythonにtype hintシステムは備わっており、mypyやpyrightなどのtype checkerを通して同様にphantom typeが実現できることを示す。

## 環境
- python3.8
- pyrightのstrict mode

## phantom typeの簡単な解説
phantom typeとはintやstrのような組み込み型とは違い、コンパイルタイム(Pythonならtoolによるtype check時)にのみ影響を与えるような型をプレースホルダ的に使用することで実現する。
型による検査を使用することでプログラム内のある種の欠陥を実行時にではなくプログラミング中に発見するテクニックである。

この説明だけではわかりにくいと思うので実際のユースケースを通じて何を目的に、それをどう実現するかを見ていく。

## 例と説明
### やりたいこと
httpリクエストで何らかのデータをpostしたい。
postなのでリクエスト先のurlとbodyが必要。

```python
def post(url: str, body: Dict[str, Any]):
  ...
```

例えば`/users`というurlには`{name: str, age: int}`、`/articles`というurlには`{title: str, body: str}`という形のbodyを送りたい。
ここで起きる問題はurlとそのbodyの形の整合性を取ることである。
間違った対応のurlとbodyを弾くためにif文などでゴチゴチに実行時検査してやるのが普通だが、phantom typeを使用するとそれを型によって静的検査で宣言的に弾く事ができる。

### 型によるpostのモデリング
postにはurlとbodyという２つの値が出てくるがそれをまずモデリングする。

```python
Body = TypeVar("Body", bound=TypedDict)

class Url(str, Generic[Body]):
  pass
```

urlの実態は`str`なので継承させるだけで他には何も実装しない。
ただし一つ違うのが`Generic[_]`を継承させることで型パラメタを取ることが可能な型として宣言する。(この場合は`Body`というパラメタ名で多相化してある。)
この型パラメタはプログラムの実行には何も影響を与えず単なるプレースホルダ=何かしらのマーキングとしてのみ振る舞う。
今回はurlに対応するbodyの型をマーキングするために使用される。
また`Body`は`dict`に厳格に型をつけるために`TypedDict`を上限境界として持つ。

続いてUrlを実際のurlごとにインスタンス化する。

```python
UsersBody = TypedDict("UsersBody", name=str, age=int)
ArticlesBody = TypedDict("ArticlesBody", title=str, body=str)

UsersUrl: Url[UsersBody] = Url("/users")
ArticlesUrl: Url[ArticlesBody] = Url("/articles")
```

これが意味するのは`UsersUrl`を使用する際には`UsersBody`型のdictをbodyにしなければならないということを示している。
`ArticlesUrl`についても同様。

urlとbodyについてのマーキングが済んだのでこれを使用して実際にpostを実装してみる。

```python
B = TypeVar("B", bound=TypedDict)

def post(url: Url[B], body: B):
  # implemantations
  ...
```

このメソッドのシグネチャは`Url[B]`のときには同時にbodyとして`B`も渡さなければないということを宣言している。
つまり先程用意したurlとbodyの対応付けがここで生きてくる。

実際に`post`をtype checkしてみる。

```python
def main():
  post(UsersUrl, {"name": "john", "age": 20}) # => ok
  post(ArticlesUrl, {"title": "dialy", "body": "hello"}) # => ok
  post(UsersUrl, {"name": "john", "age": "hello"}) # => bad
  post(UsersUrl, {"title": "dialy", "body": "hello"}) # => bad
  post(ArticlesUrl, {"title": "dialy"}) # => bad
```

type checkerで検査するとちゃんと対応として宣言された`dict`のみを受け付けるようになっている。

最後に全コード例。

```python
from typing import TypeVar, Generic, TypedDict

Body = TypeVar("Body", bound=TypedDict)

class Url(str, Generic[Body]):
  pass


UsersBody = TypedDict("UsersBody", name=str, age=int)
ArticlesBody = TypedDict("ArticlesBody", title=str, body=str)

UsersUrl: Url[UsersBody] = Url("/users")
ArticlesUrl: Url[ArticlesBody] = Url("/articles")

B = TypeVar("B", bound=TypedDict)

def post(url: Url[B], body: B):
  # implemantations
  ...


def main():
  post(UsersUrl, {"name": "john", "age": 20}) # => ok
  post(ArticlesUrl, {"title": "dialy", "body": "hello"}) # => ok
  post(UsersUrl, {"name": "john", "age": "hello"}) # => bad
  post(UsersUrl, {"title": "dialy", "body": "hello"}) # => bad
  post(ArticlesUrl, {"title": "dialy"}) # => bad
```

## おわり
Pythonもちゃんとやればちゃんとなる。
