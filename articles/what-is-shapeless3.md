---
title: "shapeless3とTypeclass Derivation"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala", "関数型プログラミング", "型クラス", "shapeless"]
published: false
---

## 始めに
Scala3に合わせてshapeless3というライブラリが登場しています。
shapelessはもともとScalaに存在しているライブラリで広く使用されています。
shapeless3はもとのshapelessとどう違うのでしょう。

この記事では次の内容の理解を目指します。
- shapeless3でできること
- shapeless2と何が違うのか
- Scala3組込みTypeclass Derivationシステムと何が違うのか

## そもそもshapelessってどんなライブラリ
ここでは3でなくもともとのshapelessがどのような働きをするライブラリなのかを概観します。

### 概説
まずはREADMEの説明を引用します。
https://github.com/milessabin/shapeless

> shapeless is a type class and dependent type based generic programming library for Scala.

ここに書かれているように、shapelessは依存型を利用してジェネリックなプログラミングスタイルを実現するライブラリです。

依存型とは値に依存した型のことです。
Scalaではdependent method typeを利用して依存型を実現します。
詳しくは[このドキュメント](https://docs.scala-lang.org/scala3/book/types-dependent-function.html)をしてください。
shapelessでは型レベルの計算を表現するときに活躍します。

またこの場合の「ジェネリックなプログラミング」とはADT（sealed trait/case class/case object/scala3のenum）に対するプログラミング程度に考えるとよいです。
たとえば通常のプログラミングではDogやCatなどの具体型に対して処理を書くことになりますが、ジェネリックなプログラミングではADTであるという知識だけを用いてプログラミングします。
逆にいえば、ADTであるという知識をプログラミングに落とし込めるものがジェネリックなプログラミングスタイルといえるでしょう。

以降の解説では簡単のため、主にcase classに対してのジェネリックプログラミングについて見ていきます。

### shapelessの用途
さまざまな機能がありますが、よく使われる機能には次のようなものがあります。
- typeclass derivation
- シングルトン型の導入
- etc

他の機能については次のドキュメントによくまとまっています。
https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0

個人的には[polynomial function](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#polymorphic-function-values)やscala3のtupleの元になったような[HList](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#heterogenous-lists)の実装があったり、果ては[case classをアラカルト形式で実装](https://github.com/milessabin/shapeless/blob/main/examples/src/main/scala/shapeless/examples/alacarte.scala)できたりと、Scalaの実験場のような印象があります。

shapelessに依存している（いた）ライブラリとしてcirce、doobieやrefinedなど数えればきりがないです。

## まずはshapeless3の使用例を眺めてみよう
何はともあれどんなことができるのか[READMEの例](https://github.com/typelevel/shapeless-3)を見てみましょう。
次の例は不要な部分を少し削っています。

```scala
import shapeless3.deriving.*

trait Monoid[A]:
  def empty: A
  def combine(x: A, y: A): A
  extension (x: A) def |+| (y: A): A = combine(x, y)

object Monoid:
  given Monoid[Boolean] with
    def empty: Boolean = false
    def combine(x: Boolean, y: Boolean): Boolean = x || y

  given Monoid[Int] with
    def empty: Int = 0
    def combine(x: Int, y: Int): Int = x+y

  given Monoid[String] with
    def empty: String = ""
    def combine(x: String, y: String): String = x+y

  given monoidGen[A](using inst: K0.ProductInstances[Monoid, A]): Monoid[A] with
    def empty: A =
      inst.construct([t] => (ma: Monoid[t]) => ma.empty)
    def combine(x: A, y: A): A =
      inst.map2(x, y)([t] => (mt: Monoid[t], t0: t, t1: t) => mt.combine(t0, t1))

  inline def derived[A](using gen: K0.ProductGeneric[A]): Monoid[A] = monoidGen

// ADT definition
case class ISB(i: Int, s: String, b: Boolean) derives Monoid
val a = ISB(23, "foo", true)
val b = ISB(13, "bar", false)

val c = a |+| b // == ISB(36, "foobar", true)

```

コードを少しずつ読んでいきましょう。
まずはtypeclassであるMonoidとともにextensionメソッドを定義しています

```scala
trait Monoid[A]:
  def empty: A
  def combine(x: A, y: A): A
  extension (x: A) def |+| (y: A): A = combine(x, y)
```

ゼロ値と結合の演算を型Aに与えます。
catsなどを使用していると馴染み深いかもしれません。
extension句は今までimplicit classやimplicit conversionを使用して実現していたことを簡易に実現する機構です。

https://dotty.epfl.ch/docs/reference/contextual/extension-methods.html

そしてコンパニオンオブジェクトではいくつかの型クラスのインスタンスを定義しています。

```scala
object Monoid:
  given Monoid[Boolean] with
    def empty: Boolean = false
    def combine(x: Boolean, y: Boolean): Boolean = x || y

  given Monoid[Int] with
    def empty: Int = 0
    def combine(x: Int, y: Int): Int = x+y

  given Monoid[String] with
    def empty: String = ""
    def combine(x: String, y: String): String = x+y
```

そして最も重要な部分がジェネリックなインスタンスを定義してそれをderivedに渡している部分です。
derivedメソッドはScala3に加わったTypeclass derivationのためのメソッドです。
詳しくは次の記事を参照してください。

https://dotty.epfl.ch/docs/reference/contextual/derivation.html

```scala
object Monoid:
  ...
  given monoidGen[A](using inst: K0.ProductInstances[Monoid, A]): Monoid[A] with
    def empty: A =
      inst.construct([t] => (ma: Monoid[t]) => ma.empty)
    def combine(x: A, y: A): A =
      inst.map2(x, y)([t] => (mt: Monoid[t], t0: t, t1: t) => mt.combine(t0, t1))

  inline def derived[A](using gen: K0.ProductGeneric[A]): Monoid[A] = monoidGen
```

このジェネリックなインスタンスを暗黙に使用することで、case classのインスタンスを導出できます。
例ではInt/String/Booleanのフィールドをもつcase class ISBに対してのインスタンスが自動導出されています。

```scala
case class ISB(i: Int, s: String, b: Boolean) derives Monoid
val a = ISB(23, "foo", true)
val b = ISB(13, "bar", false)

val c = a |+| b // == ISB(36, "foobar", true)
```

このようにshapeless3を使用することでTypeclass derivationを行うことができました。
しかし肝心のderiveのしくみがよくわかっていません。
このしくみを理解することを目標に少し遠回りします。

## 標準機能のみを用いたderivation
まずどのようなロジックで自動導出ができるかを理解するために、Scala3に組み込まれているderivation機能だけを用いたMonoidインスタンスを導出します。

型クラスの定義は変わりません。

```scala
trait Monoid[A]:
  def empty: A
  def combine(x: A, y: A): A
  extension (x: A) def |+| (y: A): A = combine(x, y)
```

インスタンス定義も変わりません。

```scala
object Monoid:
  def apply[T](using m: Monoid[T]): Monoid[T] = m

  given Monoid[Boolean] with
    def empty = false

    def combine(x: Boolean, y: Boolean): Boolean = x || y

  given Monoid[Int] with
    def empty = 0

    def combine(x: Int, y: Int): Int = x + y

  given Monoid[String] with
    def empty = ""

    def combine(x: String, y: String): String = x + y
```

そしてderived部分は次のようになります。

```scala
object Monoid:
  inline def summonAll[T <: Tuple]: List[Monoid[?]] =
    inline erasedValue[T] match
      case _: EmptyTuple => Nil
      case _: (t *: ts)  => summonInline[Monoid[t]] :: summonAll[ts]


  inline given derived[A](using m: Mirror.ProductOf[A]): Monoid[A] = new Monoid[A]:
    override def empty: A =
      val instances = summonAll[m.MirroredElemTypes]
      m.fromProduct(Tuple.fromArray(instances.map(_.empty).toArray))

    override def combine(x: A, y: A): A =
      val instances = summonAll[m.MirroredElemTypes]
      val xs             = x.asInstanceOf[Product].productIterator
      val ys             = y.asInstanceOf[Product].productIterator
      val combineds      = xs.zip(ys).zipWithIndex.map { case ((xx, yy), i) =>
        instances(i).asInstanceOf[Monoid[Any]].combine(xx, yy)
      }

      m.fromProduct(Tuple.fromArray(combineds.toArray))
```

以上を用いるとshapeless3の実装と同じことが実現できます。
少しずつ見ていきます。

### emptyの実装
まずはemptyの実装について確認しましょう。
ここでのcase classに対してemptyは、各フィールドの型のempty値をフィールドにもつようなcase classのインスタンスが定義できれば理想です。
つまり次のようになってほしいです。

```scala
summon[Monoid[ISB]].empty => ISB(0, "", false)
```

このようなemptyの実装は次のようになります。

```scala
override def empty: A =
  val instances = summonAll[m.MirroredElemTypes]
  m.fromProduct(Tuple.fromArray(instances.map(_.empty).toArray))
```

まずはsummonAllという関数を呼び出ます。
これは型パラメータに突っ込んだTuple型のすべての要素に対してMonoidインスタンスをsummonしてListにして返すような関数です。
この関数は公式からほとんどそのまま持ってきています。

https://dotty.epfl.ch/docs/reference/contextual/derivation.html#type-classes-supporting-automatic-deriving

```scala
inline def summonAll[T <: Tuple]: List[Monoid[?]] =
  inline erasedValue[T] match
    case _: EmptyTuple => Nil
    case _: (t *: ts)  => summonInline[Monoid[t]] :: summonAll[ts]
```

`m.MirroedElemTypes`というのは、対象のcase classのフィールドの型を要素にもつタプルです。
たとえばISBの場合には`Int *: String *: Boolean *: EmptyTuple`になります。

Scala3のTupleは2系と大幅に異なるので、適宜以下を参照してください。

https://dotty.epfl.ch/api/scala/Tuple.html


summonAllを使用することでフィールドのMonoidインスタンスのListを取得できたので、各empty値を取得できます。

```scala
instances.map(_.empty)
```

MirrorオブジェクトはProduct型からMirrorが表す元の型に戻すための関数`fromProduct`を持っているので、それを使用して対象のcase classの値を得ます。

```scala
m.fromProduct(Tuple.fromArray(instances.map(_.empty).toArray)) 
```
empty値はこのようにして得られます。

### combineの実装
次はcombineです。

まずはemptyのときと同じように各フィールドのMonoidインスタンスを取得します。

```scala
val instances = summonAll[m.MirroredElemTypes]
```

次にcombineの右辺・左辺のフィールドの値を抜き出します。

```scala
val xs = x.asInstanceOf[Product].productIterator
val ys = y.asInstanceOf[Product].productIterator
```

そしてiteratorをzipして各フィールドの値をMonoidインスタンスを使用してcombineします。

```scala
val combineds = xs.zip(ys).zipWithIndex.map { case ((xx, yy), i) =>
  instances(i).asInstanceOf[Monoid[Any]].combine(xx, yy)
}
```

ちょっと複雑ですが、イメージとしては次のような感じです。

```scala
(1, "a", false) 
                 -zip-> ((1, 2), ("a", "b"), (false, true)) -combine-> (3, "ab", true)
(2, "b", true)
```

最後に再び`m.fromProduct`を用いて対象の型の値を得ます。

```scala
m.fromProduct(Tuple.fromArray(combineds.toArray))
```

### 問題点
以上のようにshapeless3を使用しなくても標準機能だけで十分typeclass derivationが可能です。
しかし次のような問題点が存在します。

1. shapeless3を使用した場合に比べて冗長
2. 型がほとんど役に立っていない

特に2は致命的で、対象型クラスが複雑になればなるほどScalaの型システムを無視したカオスなコードになると考えられます。
型がないことで当然間違えたロジックを書くことも可能になりますし、プログラマーが頭を使わなければならない場所も増えます。

このようにTypeclass Derivationを型安全にするのは技術的な課題があります。
次はどのような機構が存在すればこの課題を解決できるかを、shapeless2系から学びます。

## shapeless2系で実装してみる
さっそくですが、実装は次のようになります。
重要なのがこれらはすべて型安全です。

```scala
import shapeless.ops.hlist.ZipWith
import shapeless.{::, Generic, HList, HNil, Poly2}

trait Monoid[A] {
  def empty: A
  def combine(x: A, y: A): A

}

object Monoid {
  implicit class ops[A](a: A)(implicit m: Monoid[A]) {
    def |+|(y: A): A = m.combine(a, y)
  }

  implicit object intMonoid extends Monoid[Int] {
    override def empty: Int = 0

    override def combine(x: Int, y: Int): Int = x + y
  }

  implicit object stringMonoid extends Monoid[String] {
    override def empty: String = ""

    override def combine(x: String, y: String): String = x + y
  }

  implicit object boolMonoid extends Monoid[Boolean] {
    override def empty: Boolean = false

    override def combine(x: Boolean, y: Boolean): Boolean = x || y
  }

  object combinePoly extends Poly2 {
    implicit def caseT[T](implicit mon: Monoid[T]): Case.Aux[T, T, T] = at[T, T](mon.combine)
  }

  implicit object hnilMonoid extends Monoid[HNil] {
    override def empty: HNil = HNil

    override def combine(x: HNil, y: HNil): HNil = HNil
  }

  implicit def hconsMonoid[H, T <: HList](implicit
      hmon: Monoid[H],
      tmon: Monoid[T]
  ): Monoid[H :: T] =
    new Monoid[H :: T] {
      override def empty: H :: T = hmon.empty :: tmon.empty

      override def combine(x: H :: T, y: H :: T): H :: T =
        hmon.combine(x.head, y.head) :: tmon.combine(x.tail, y.tail)
    }

  implicit def derived[T, Repr <: HList](implicit
      gen: Generic.Aux[T, Repr],
      mon: Monoid[Repr],
      zipWith: ZipWith.Aux[Repr, Repr, combinePoly.type, Repr]
  ): Monoid[T] =
    new Monoid[T] {
      override def empty: T = gen.from(mon.empty)

      override def combine(x: T, y: T): T = gen.from(gen.to(x).zipWith(gen.to(y))(combinePoly))
    }
}
```

また少しずつ見ていきましょう。

型クラスのインスタンス定義はシンタックスを除いてほとんど変わりません。

```scala
trait Monoid[A] {
  def empty: A
  def combine(x: A, y: A): A
}
```

ただextensionメソッドの定義がimplicit classになったのが少し違う点です。

```scala
object Monoid {
  implicit class ops[A](a: A)(implicit m: Monoid[A]) {
    def |+|(y: A): A = m.combine(a, y)
  }
  ...
}
```

各種型クラスインスタンスの定義です。

```scala
object Monoid {
  ...
  implicit object intMonoid extends Monoid[Int] {
    override def empty: Int = 0

    override def combine(x: Int, y: Int): Int = x + y
  }

  implicit object stringMonoid extends Monoid[String] {
    override def empty: String = ""

    override def combine(x: String, y: String): String = x + y
  }

  implicit object boolMonoid extends Monoid[Boolean] {
    override def empty: Boolean = false

    override def combine(x: Boolean, y: Boolean): Boolean = x || y
  }
  ...
}
```

次からが重要な部分です。
ある一定の制約を満たす型Tに対してMonoidインスタンスを定義します。

```scala
implicit def derived[T, Repr <: HList](implicit
    gen: Generic.Aux[T, Repr],
    mon: Monoid[Repr],
    zipWith: ZipWith.Aux[Repr, Repr, combinePoly.type, Repr]
): Monoid[T] =
  new Monoid[T] {
    override def empty: T = gen.from(mon.empty)

    override def combine(x: T, y: T): T = gen.from(gen.to(x).zipWith(gen.to(y))(combinePoly))
  }
```

Tに対する成約は大きく3つで、implicitパラメータの数に対応します。

１つ目は`Generic.Aux[T, Repr]`のインスタンスが定義されていることです。
Genericについてここでは詳しくは解説しませんが、HListにcase classを還元するための型だと考えられます。
つまりTのHList表現がReprだと読み替えられるということです。
HListとはcase classのフィールド型を要素にもつ拡張可能なタプルのようなものです。
たとえば`ISB`に対応するHListの型は`Int :: String :: Boolean :: HNil`になります。

詳しくは次のドキュメントを参照してください。

https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0

２つ目は`Monoid[Repr]`インスタンスが定まっている、つまりRepr（Tを表すHList）型に対してMonoidが定まっていることです。

３つ目は`ZipWith.Aux[Repr, Repr, combinePoly.type, Repr]`インスタンスが存在することですが、これについては後述します。

次は制約を満たしたうえでのインスタンスの実際の定義について見ていきます。

### emptyの実装

emptyの実装は次のようになります。

```scala
override def empty: T = gen.from(mon.empty)
```

まず`Monoid[Repr]`型のMonoidインスタンスmonがあるので、そのempty値を取得します。
それを`Generic.Aux[T, Repr]`型のGenericインスタンスgenを使って`gen.from`を使って対象の型Tに戻します。
これでemptyが定義できました。

### combineの実装

次はcombineの実装です。

```scala
override def combine(x: T, y: T): T = gen.from(gen.to(x).zipWith(gen.to(y))(combinePoly))
```

まずはgenを使ってT型の変数xとyをHList表現であるReprに変換します。

```scala
gen.to(x) // => Repr
gen.to(y) // => Repr
```

これはたとえば`T =:= ISB`の場合は`Repr =:= Int :: String :: Boolean :: HNil`となります。

次はこの２つのRepr型の値をzipWith関数とcombinePolyを使用してRepr型のHListを作成します。
ここから一気に複雑になるのでひとつずつ説明します。

まずやりたいことを整理すると、結局はcombine操作を各フィールドに適用することです。
case classはHListとして表現されている状況で、 つまり次のようなことがやりたいです。

```scala
gen.to(x) = 1 :: "a" :: false :: HNil
                                     -zip-> (1, 2) :: ("a", "b") :: (false, true) :: HNil -combine-> 3 :: "ab" :: true :: HNil
gen.to(y) = 2 :: "b" :: true :: HNil
```

こう見ると、Scala3の標準機能を使用したときとやりたいこと自体は変わっていないです。
これを一気にやってくれるHListのオペレータとしてzipWithというものがあります。
zipWithは２つのHListを各要素でzipしてから、zipした各ペア（`(1, 2)`など）を処理する関数を次々に適用していきます。
つまり、この関数を`p`とすればpは２つの引数を受け取って値をひとつ返す関数です。

ここでひとつ問題があります。
関数pはmonomorphic、つまり関数インスタンスの実態としてひとつの型に対する処理しか実行できません。
難しいのでもっと噛み砕いていうと、たとえばひとつの関数でStringに対する処理とIntに関する処理を同時に扱うことが（真にジェネリックな方法では）できないということです。
当たり前のことを言っていますが、もう少し詳しく見ます。

次のような例ではStringとIntに対するどちらかの処理しか行えないことは明らかです。
これをmonomorphicといいます。

```scala
def handleString(x: String, y: String): String = x + y
def handleInt(x: Int, y: Int): String = x + y
```

これをジェネリック、つまりpolymorphicにしたいときまず思い付くのは型パラメータを導入することでしょう。

```scala
def handle[T](x: T, y: T): T = ???
```

しかし右辺の処理はどのように書けばいいのでしょうか。
IntとStringという具体型をTにしたことで、Tに対してどのような操作ができるかという情報が消えました。
そのため、実際にこのままジェネリックな方法で処理を書くことはできません。
パターンマッチで対象の型に対する処理を連ねていくことはできますが、Tが任意の型になりうるということを考えるとそれは現実的でありません。

このようなTに対する情報がなく、何が行えるかの情報がほしいときに活躍するのが型クラスです。
shapelessは任意の引数型に対して、polymorphicに処理を記述できる機構としてpolymorphic functionを提供しています。
たとえばIntやString、Tuple2に対してsizeを処理するpolymorphic functionは次のように記述できます。
（公式からの引用）

https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#polymorphic-function-values

```scala
object size extends Poly1 {
  implicit def caseInt = at[Int](x => 1)
  implicit def caseString = at[String](_.length)
  implicit def caseTuple[T, U]
    (implicit st : Case.Aux[T, Int], su : Case.Aux[U, Int]) =
      at[(T, U)](t => size(t._1)+size(t._2))
}

scala> size(23)
res4: Int = 1

scala> size("foo")
res5: Int = 3

scala> size((23, "foo"))
res6: Int = 4

scala> size(((23, "foo"), 13))
res7: Int = 5
```

Poly1というのは関数の引数がひとつということです。
implicitでatを使って各引数型に対しての処理を書けば、それが各型に対しての処理内容になります。

ここではこれ以上詳しくは語りませんが、要は任意の型に対して異なる振る舞いをする関数がほしいということです。
より詳細に知りたい人は、次の記事を参照してください。
- http://milessabin.com/blog/2012/04/27/shapeless-polymorphic-function-values-1/
- https://milessabin.com/blog/2012/05/10/shapeless-polymorphic-function-values-2/

こうしてpolymorphicな関数を手に入れることができました。
もともとやりたかったことを思い出すと、任意のT型の２つの引数をとってcombineするような関数pがほしいということです。
そのようなpolymorphicな関数の定義は次のようになります。

```scala
object combinePoly extends Poly2 {
  implicit def caseT[T](implicit mon: Monoid[T]): Case.Aux[T, T, T] = at[T, T](mon.combine)
}
```

Poly2というのは関数の引数が２つということです。
at[T, T]となっているのは、引数が２つなのでそのどちらもがT型だということを明示しています。
あとは処理として、Monoid[T]型のインスタンスが定まっていればそれを使用してcombineするというような内容になっています。
Tに対して一般化しているので少し分かりづらいかもしれません。
もしString/Int/Booleanに対しての処理のみを定義したい場合は次のようになります。

```scala
object combinePoly extends Poly2 {
  implicit def caseString(implicit mon: Monoid[String]): Case.Aux[String, String, String] = at[String, String](mon.combine)
  
  implicit def caseInt(implicit mon: Monoid[Int]): Case.Aux[Int, Int, Int] = at[Int, Int](mon.combine)
  
  implicit def caseBoolean(implicit mon: Monoid[Boolean]): Case.Aux[Boolean, Boolean, Boolean] = at[Boolean, Boolean](mon.combine)
}
```

このcombinePolyを使うことで、zipWith処理を記述できます。

```scala
gen.to(x).zipWith(gen.to(y))(combinePoly) // => Repr型
```

ここでderivedメソッドの３つ目の制約である`ZipWith.Aux[Repr, Repr, combinePoly.type, Repr]`について思い出します。
これは２つのRepr型のHListをzipしてからcombinePolyで各ペアを処理して、最終的にはRepr型のHListになることを表明しています。
zipWithメソッドがimplicitにこのインスタンスを要求しているので、このような処理が可能かどうかはコンパイル時にチェックされます。
つまり仮に`caseInt`、`caseString`しかない場合には`implicitly[Monoid[ISB]]`はコンパイルエラーとなるということです。

最後は`gen.from`でRepr型の値をTに変換するだけです。

```scala
gen.from(gen.to(x).zipWith(gen.to(y))(combinePoly)) // => T型
```

こうしてcombineの実装ができました。

### 型安全なTypeclass Derivationに必要な機構の整理
上記の実装例から何が型安全なTypeclass Derivationに必要かを考察します。
まず手順を簡単に概観すると次のようになります。

1. 対象のT型の値をRepr型のHListに変換する
2. HListに対しての処理をpolymorphic functionを使用して記述する
3. 処理が終わったHListを何らかの型（この場合は再びT型）の値に戻す

このように見ると２つの大きな機構が存在していることがわかります。

１つ目はHListです。
これはADTのProduct型にあたるものを型安全に表現する機構です。
HListは任意のcase classをひとつのデータ型で記述できるプラットフォームのようなものです。

２つ目はpolymorphic functionです。
これによって型安全かつ抽象的に表現されたHListに対しての処理も型安全に記述できます。

この整理を前提にして、再びshapeless3によって何ができるかを見てみます。

## shapeless3のScala3における役割
前節で整理したとおり、type safeなTypeclass DerivationにはHListとpolymorphic functionのようなものが必要だとわかりました。

まずHListについて考えます。
Scala3にはTupleがHListのような役割を担うことができます。
2系のTupleはただの値のコンテナであり、そこに定義された操作も貧弱でした。
しかし3のTupleは拡張やmapなどの処理が定義されており、十分HListとして振る舞えます。
あとはTupleをフィールドの集まりとしてコンパイルタイムに持ち込む方法ですが、Scala3の標準機能での実装の節で見たとおりMirrorがその役割を果たします。
実際にMirror.ProductOfにはフィールドの型がMirroredElemTypesとしてTupleで定義されています。

```scala
type ProductOf[T] = Mirror.Product { type MirroredType = T; type MirroredMonoType = T; type MirroredElemTypes <: Tuple }
```

## 参考
- https://youtu.be/CFyypCbLRAo
- https://youtu.be/yUzcQJpbT2k
