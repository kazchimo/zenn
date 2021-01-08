---
title: "ZIOへの環境Rのprovide方法各種"
emoji: "💲️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [scala, zio]
published: false
---

## はじめに
ScalaのライブラリZIOにはDIを扱うための仕組みが組み込みで備わっている。
DIの実態をprovideする方法はいくつかあるので紹介する。
なおZIOの基本的な理解は所与とする。
DIを補助する仕組みとしての`ZLayer`や`Has`については[この記事](https://takatorix.hatenablog.com/entry/2020/12/23/230934)がよくまとまっている。

## provide
`ZIO#provide`メソッドは`ZIO[R, _, _]`として依存している`R`をすべて解決する。

```scala
trait Logger { ... }
trait DB { ... }

val z: RIO[Logger with DB, Unit] = ...
val env = new Logger with DB { ... }

z.provide(env)
// => ZIO[Any, Throwable, Unit]
```

依存は`Any`となりどのコンポーネントにも依存しない実行可能な`ZIO`が生成される。

ちなみにすでに依存が存在しないprovideする必要のない`ZIO[Any, _, _]`に対して`provide`を呼ぶとコンパイルが通らない。
よくできている。
仕組みは簡単で、`NeedsEnv[T]`というimplicit parameterを`provide`メソッドに付与し、`Any`に対してのみわざと２つの同じ優先度を持つインスタンスを配置することで
implicit ambiguousを起こしている。

```scala
sealed trait ZIO[-R, +E, +A] extends Serializable with ZIOPlatformSpecific[R, E, A] { self =>
  def provide(r: R)(implicit ev: NeedsEnv[R]): IO[E, A] = ...
}

sealed abstract class NeedsEnv[+R]

object NeedsEnv extends NeedsEnv[Nothing] {
  // R =:= Any以外のときには問答無用でインスタンスを提供する
  implicit def needsEnv[R]: NeedsEnv[R] = NeedsEnv

  // R =:= Anyのときにはわざと２つインスタンスを配置しimplicit ambiguous 
  implicit val needsEnvAmbiguous1: NeedsEnv[Any] = NeedsEnv
  implicit val needsEnvAmbiguous2: NeedsEnv[Any] = NeedsEnv
}
```

## provideSome
前述の`provide`メソッドでは依存`R`を一気に解決する必要があったが、部分的に解決できたほうが便利なことも多い。
`provideSome`を使用することで部分的な解決が可能になる。

```scala
val z: RIO[Logger with DB, Unit] = ...

z.provideSome[DB](db =>
  new Logger with DB {
    // implementations using given DB instance
  }
)
// => ZIO[DB, Throwable, Unit]
```

`DB`を与えられたと仮定した場合のすべての依存の実装を提供することで`DB`にのみ依存する`ZIO`が作られる。
注意しなければならないのが型パラメータで`provideSome[DB]`のように依存させるサービスを宣言しなければならない。

また`provide`のように`ZIO[Any, _, _]`に対して`provideSome`を呼んでもコンパイルが通らない。

## provideLayer
zioで依存を管理するときは`ZLayer`を使ったmoduleパターンで設計することが[典型](https://github.com/zio/zio/discussions/4526#discussioncomment-249489)。
`provideLayer`を使用することで`ZLayer`を`ZIO`に依存として提供することができる。

```scala
val z: RIO[Has[Logger] with Has[DB], Unit] = ...
val layer = ZLayer.succeedMany(Has.allOf(new Logger { ... }, new DB { ... }))

z.provideLayer(layer)
// => ZIO[Any, Throwable, Unit]
```

ただし一気にすべてのコンポーネントを提供しなければならない。

## provideSomeLayer
`provideSome`のように部分的に`ZLayer`の依存を解決できる。

```scala
val z: RIO[Has[Logger] with Has[DB], Unit] = ...
val layer = ZLayer.succeed(new Logger { ... })

z.provideSomeLayer[Has[DB]](layer)
// => ZIO[Has[DB], Throwable, Unit]
```

`provideSome`の様に解決後も残ってしまう依存の型をパラメタで宣言しなければならない。

## provideCustomLayer
`provideSomeLayer`の特化版。
依存解決後も残ってしまう型が`ZEnv`に固定されたもの。
`zio.App`traitを使用したzioアプリケーションの実行時には`ZEnv`の依存はデフォルト値が与えられるため、そのようなときに有用。

```scala
val z: RIO[Has[Logger] with Has[DB] with ZEnv, Unit] = ...
val layer = ZLayer.succeedMany(Has.allOf(new Logger { ... }, new DB { ... }))

z.provideCustomLayer(layer)
// => ZIO[ZEnv, Throwable, Unit]
```

`ZEnv`以外の依存はすべて与えなければならない。

## おわり
zioきもちい。



