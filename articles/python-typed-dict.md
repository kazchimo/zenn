---
title: "TypedDictでpythonのtype hintでdictのkeyとvalueに厳格に型をつける"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python, 型, typing]
published: true
---

## はじめに
pythonのtype hintでdict型のkeyとvalueの対応に厳格に型をつける方法を紹介する。

## 動作環境
- pythonのtype checkにはpyrightのstrict modeを使用(多分他のtype checkツールでも同じ結果、試してないけど)
- python3.8

## 組み込み`Dict`の問題点
組み込みのDict typeには厳格にkeyとvalueの対応付けを考慮した型付けができない。
例えば以下の様な関数があったとする。

```python
def show_animal(a):
    print(a["name"])
    print(a["age"])
```

この関数は`name`と`age`keyを持ちかつ`name`として`int`を、`age`として`int`をもつ`dict`のみを受け取りたい。
keyだけの制限なら`Literal`型を使用すれば制約をかけられる。

```python
Animal = Dict[Literal["name", "age"], Union[str, int]]

def show_animal(a: Animal):
    print(a["name"])
    print(a["age"])

show_animal({"name": "taro", "age": 5}) # ok
show_animal({"name": "taro", "color": "black"}) # bad
```

しかしこの方法の問題点としてkeyとvalueの対応付けまでは制限をかけられない。
具体的には以下が通ってしまう。

```python
show_animal({"name": 5, "age": "taro"}) # ok
```

これを通したくない。

## classを用いた解決
class、特に直積型のコンテナとして優秀な`dataclass`を使用すればkeyとvalueの型の対応付けが保証できる。

```python
@dataclass
class Animal:
    name: str
    age: int
    
def show_animal(a: Animal):
    print(a["name"])
    print(a["age"])
    

show_animal(Animal("taro", 5)) # ok
show_animal(Animal(5, "taro")) # bad
```

しかしクラスの定義はいいとして関数に渡すときに該当のインスタンスを生成する必要がある。
dictのまま扱いたいときにはこのやり方はそぐわない。

## TypedDictを用いた解決
dictのまま厳格に型を付けたいusecase用に`TypedDict`という便利なものが用意されている。

```python
class Animal(TypedDict):
    name: str
    age: int

def show_animal(a: Animal):
    print(a["name"])
    print(a["age"])


show_animal({"name": "taro", "age": 5}) # ok
show_animal({"name": "taro", "color": "black"}) # bad
show_animal({"name": 5, "age": "taro"}) # bad
```

これで望み通りにkeyとvalueの対応に対して型が付けられる。
ちなみに`TypedDict`を用いた定義は継承以外にも用意されている。

```python
Animal = TypedDict("Animal", name=str, age=int)
Animal = TypedDict("Animal", {"name": str, "age": int})
```

## 最後に
pythonのtype hint(とチェックツールを用いた静的型検査)はやればまあまあできる子。
