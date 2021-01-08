---
title: "代数的データ型Deep Dive in Scala"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala", "関数型プログラミング", "代数的データ型", "ADT"]
published: false
---


## はじめに
関数型やってるとよく聞く代数的データ型(ADT)について。
Scalaやってるとよく使うがHaskellほどこの話題について掘った記事はあんまりないのでdigっていきたい。

## そもそも代数的データ型とは
### 具体例
こういうやつ。

```scala
sealed trait Animal
case class Dog(name: String, age: Int) extends Animal
case class Cat(color: String) extends Animal
```

### なぜ代数的データ型というのか
データ型をある種代数=計算対象とその上の演算・法則に対応させることができるからである。
例えば代数的データ型として定義されたものは足し算や掛け算の様なものに対応させることができる。
詳しくは後述する。


## 代数型データ型の構成要素
## 直和型
下のようなもので表されるもの。

```scala
sealed trait A
case class B(v: Int) extends A
case object C extends A
```

ここで`A`という型としてありえる具体型は、`B`または`C`とみなせる。
なぜなら他のファイルでは`A`は継承できないし、`new A {}`のように直接インスタンス化することもできない。
(同一ファイル内で`new A {}`することはできるっぽいがそれはなしの方向で)
故に`A`としてありえる値のパターン数は「`B`としてありえる値のパターン数+`C`としてありえる値のパターン数」に等しい。
そして`B`のパターンは内部のIntのパターン数2^32に等しく`C`は唯一`C`だけがありえる値である。
結果として型に対応するありえる値の数を二重線文字で書くならば、

$$
\begin{aligned}
{\Bbb A} &= \Bbb B + \Bbb C \\
        &= 2^{32} + 1
\end{aligned}
$$

となる。
Aのパターンとして具体例を挙げると、
`C`、`B(-2147483648)`、…、`B(2147483647)`
と確かに2^32 + 1個ある。

この様に直和型と言われるものの値の数はそのサブ型の値の数の和によって求められる。
代数的演算としては足し算に対応する。
しかもそのサブ型同士に被りはなく、集合論の直和集合に対応している。

これは仮に`A`としてありえる値が`B`、`C`、`D`、…と増えていっても同じである。

以下型に対応する値の数をその型の二重線文字で表現する。

## 直積型
和に加えて積がもう一つの構成要素になる。
具体的には以下のようなものである。

```scala
case class B(v: Int)
object C
case class A(b: B, c: C)
```

ここで$\Bbb A$を考えてみよう。
Aは２つの要素bとcによってなる。
つまりbの任意の要素に対してcの任意の要素の組を作ることができ、その数は$\Bbb B \times \Bbb C$に等しい。
よって、

$$
\begin{aligned}
\Bbb A &= \Bbb B * \Bbb C \\
        &= 2^{32} * 1 \\
        &= 2 ^ {32}
\end{aligned}
$$

となる。
`A`のパターンとして具体例を挙げると、
`A(B(-2147483648), C)`、…、`A(B(2147483647), C)`
と確かに2^32通りになっている。

直積型の値の数はそれをなす型の数の積によって求められる。
代数的演算としては掛け算に対応する。

これは仮に`A`をなす要素が`B`、`C`、`D`、…と増えていっても同じ様にそれらの積をすればいいだけである。

### 代数的データ型の構造
代数的データ型は結局一番最初に出した例のように直和として表現される。
そしてそのサブ型は直積であるためしばしば「直積の直和」や、「直積の総和」などと表現されることもある。


## データ型と代数
なんとなくデータ型を使った足し算と掛け算がわかったので、頻出するデータ型が代数的には何に対応するかを見ていこうと思う。

### Nothing=0
`Nothing`は値がないことを保証する型で、例外を射出するときなどに使用する。
値がないのだから0である。

### Unit=1
Scalaの`Unit`は唯一つの値`()`を持つため1に対応する。
同じ様にobjectなどのシングルトンたちも1に対応する。

### Boolean=2
`Boolean`は`true`もしくは`false`の２値。
よって2。

### Option[T]=T+1
`Option[T]`は`Some(T)`もしくは`None`である。
`Some`は単項直積でそのまま中身の値の種類に依存するため$\Bbb T$でありNoneはシングルトンであるから、

$$
\begin{aligned}
\Bbb Option[T] &= \Bbb T + 1
\end{aligned}
$$

である。

### Either[A, B]=A+B
`Either`は`Left`もしくは`Right`でありどちらも単項直積で、

$$
\Bbb Either[A, B] = \Bbb A + \Bbb B
$$

である。
これは最も単純な直和であり、代数的データ型の例としてよく見る。

また仮に$\Bbb B = 1$、例えば`B`が`Unit`であるようなときは、

$$
\begin{aligned}
\Bbb Either[A, Unit] &= \Bbb A + 1 \\
                      &= \Bbb Option[A]
\end{aligned}
$$

となりこれは`Either[A, Unit]`と`Option[A]`が同じ表現力を持つことを示している。
確かに扱いやすさを別にすれば`Right[Unit]`であることを`None`に、`Left[A]`であることを`Some[A]`に対応させれば同じことが表現できる。

他にも、`Either[A, Nothing]`なるものを考えれば、

$$
\begin{aligned}
\Bbb Either[A, Nothing] &= \Bbb A + 0 \\
                        &= \Bbb A
\end{aligned}
$$


となり単なる`A`に等しい。
`Right`は起こり得ないのだからそれもそうである。

### Tuple2[A, B]=A*B
タプルは任意の型の組み合わせを表現することができ、これは最も単純な直積である。
つまり、

$$
\Bbb Tuple2[A, B]=A*B
$$

タプルの要素が2、3、…と増えていけば掛け算の項が増えていくだけ。

また仮に`B`=`Nothing`のときは、

$$
\begin{aligned}
\Bbb Tuple2[A, Nothing] &= \Bbb A * 0 \\
                        &= 0 \\
                        &= \Bbb Nothing
\end{aligned}
$$

`Nothing`が入った瞬間に存在する値がないのだからそれはそう。

他にも`B`=`Unit`とすると、

$$
\begin{aligned}
\Bbb Tuple2[A, Unit] &= \Bbb A * 1 \\
                    &= \Bbb A
\end{aligned}
$$

これも片方を`()`に固定して`A`の値を動かしていくだけなので表現力としては同じなのは腑に落ちる。


## 代数について
ここまで結構適当に代数というワードを使用してきたが、今一度代数とはなにかという部分を見ていきたい。

### 代数の構成
まず代数をなす要素から見ていく。
「代数」とは簡単に言えば次の３つの要素からなるものである。

1. 何らかの集合A
2. 集合Aの上での演算の集まり
3. 演算が満たすべき法則

これら３つを備えたものを代数とか代数系とか呼んだりする。
またここでいう集合Aをunderlyingとか台集合とか呼んだりする。
イメージとしては集合Aが土台となって、その上で何らかの演算が定まり、それらが法則を満たしながら構成されているという感じ。

### 代数の具体例
代数の構成の定義だけ見てもなんのこっちゃわからないので具体例を見ていく。
下に上げたもの以外にも色々存在する

#### Semigroup(半群)
Catsとか使うと見たことある名前だと思う。
Semigroupは、

1. 何らかの集合Aがあって
2. その上でなんらかの二項演算子$*: A \times A \to A$が定まっていて
3. Aの任意の要素a,b,cに対して$(a*b)*c = a*(b*c)$が成り立っている(結合則)

とき、`(A, *)`の組をSemigroupと呼ぶ。
これもまた代数の具体例にしては抽象的なのでもっと具体化してみる。
Semigroupといっても実際にSemigroupになるようなものはいくつかあって、例えば自然数の集合Nとその上での足し算+の組(N, +)はSemigropuになっている。
なぜなら任意の自然数a、b、cに対して、$(a + b) + c = a + (b + c)$となるからだ。
ちなみに同様に掛け算に対してもNはSemigroupをなすが、割り算や引き算に対しては結果が自然数に収まらないのでそれらに対してはSemigroupをなさない。

#### Monoid
Semigroupとほとんど同じ。
違うのは次の定義で定まる単位元を持つかどうか。

- 単位元とは、台集合Aの任意の要素aに対して、$e*a = a*e = a$となるようなAの要素eのこと

簡単に言うとその要素と別の要素の計算がその値を変化させないようなものである。
自然数の上での足し算の場合は0、自然数の上での掛け算は1がそれに対応する。


## 代数的データ型がなす代数
代数的データ型はどんな代数的構造が現れてくるか見ていきたい。

### 直和
直和として単純な`Either[A, B]`を見ていく。
例えば、

$$
\begin{aligned}
\Bbb Either[Either[A, B], C] &= (\Bbb A + \Bbb B) + \Bbb C \\
                            &= \Bbb A + (\Bbb B + \Bbb C) \\
                            &= \Bbb Either[A, Either[B, C]]
\end{aligned}
$$

となるので`Either`は結合則を満足する。
更に、

$$
\begin{aligned}
\Bbb Either[Nothing, A] &= 0 + \Bbb A \\
                        &= \Bbb A \\
                        &= \Bbb A + 0 \\
                        &= \Bbb Either[A, Nothing]
\end{aligned}
$$

となるので`Either`は単位元`Nothing`を持つ。
加えて、

$$
\begin{aligned}
\Bbb Either[A, B] &= \Bbb A + \Bbb B \\
                    &= \Bbb B + \Bbb A \\
                    &= \Bbb Either[B, A]
\end{aligned}
$$

より`Either`は可換でもある。

以上から`Either`は可換Monoidの性質を持つことがわかる。

### 直積
`Tuple2[A, B]`を考える。
例えば

$$
\begin{aligned}
\Bbb Tuple2[A, Tuple2[B, C]] &= \Bbb A * (\Bbb B * \Bbb C) \\
                            &= \Bbb A * \Bbb B * \Bbb C \\
                            &= (\Bbb A * \Bbb B) * \Bbb C \\
                            &= \Bbb Tuple2[Tuple2[A, B], C]
\end{aligned}
$$

であるから`Tuple2[A, B]`は結合則を満足している。
更に、

$$
\begin{aligned}
\Bbb Tuple2[A, Unit] &= \Bbb A * 1 \\
                    &= \Bbb A \\
                    &= 1 * \Bbb A \\
                    &= Tuple2[Unit, A]
\end{aligned}
$$


となり`Tuple2[A, B]`は単位元`Unit`を持つ。
またゼロ値について考えてみると、

$$
\begin{aligned}
Tuple2[A, Nothing] &= \Bbb A * 0 \\
                    &= 0 \\
                    &= 0 * \Bbb A \\
                    &= \Bbb Nothing
\end{aligned}
$$

となるので、`Nothing`がゼロ値として振る舞う。

### 直積と直和
直積と直和の間の関係についても見ていく。
例えば以下の様なことが成り立つ。

$$
\begin{aligned}
\Bbb Tuple2[A, Either[B, C]] &= \Bbb A * (\Bbb B + \Bbb C) \\
                            &= \Bbb A * \Bbb B + \Bbb A * \Bbb C \\
                            &= \Bbb Either[Tuple2[A, B], Tuple2[A, C]]
\end{aligned}
$$

順番を逆にしても計算すると同様に

$$
\Bbb Tuple2[Either[A, B], C] = \Bbb Either[Tuple2[A, C], Tuple2[B, C]]
$$

であることがわかる。
この様な関係を分配法則という。

### Semiring
以上から代数的データ型は以下の性質を持つことがわかった。

1. 和(`Either[A, B]`)について可換Monoidをなす
2. 積(`Tuple2[A, B]`)についてMonoidをなす
3. 積にゼロ値が存在する
4. 積は和の上で分配的に振る舞う

以上の性質を満たす代数的構造をSemiring(半環)という。
つまり代数的データ型はSemiring構造をなす。
これが代数的データ型が`代数的`と言われる所以である。


## データ型で表されるその他の計算構造
以前の節で代数的データ型と代数についての話は終わったが、他のデータがで表される計算構造についてもついでなので見ていく。

### 関数
#### 表現
関数の型は指数に対応する。
つまり、

$$
\Bbb A => \Bbb B = \Bbb B ^ {\Bbb A}
$$

である。
これは集合論をちょっとかじった人なら見たことがあると思う。
なぜこうなるか見ていこう。
`A => B`はA型の各々の値にB型のある値を紐付ける操作だとみなすことができる。
そしてその紐付け方法は$\Bbb B$通りある。
$\Bbb B$通りの紐付け方法がAの要素分=$\Bbb A$だけ存在するから、そのバリエーションは$\Bbb B * \Bbb B * … * \Bbb B = \Bbb B ^ \Bbb A$である。

#### Unitを含んだ計算
関数表現の指数を使ってちょっと遊んで見る。

まず、

$$
\begin{aligned}
\Bbb A => Unit &= 1 ^ \Bbb A \\
                &= 1 \\
                &= \Bbb Unit \\
\end{aligned}
$$

となる。
どんな`A`に対しても`Unit`を返すならば、それは表現力としてはただの`Unit`と変わりないということだ。

逆にして考えてみると、

$$
\begin{aligned}
\Bbb Unit => A &= \Bbb A ^ 1 \\
            &= \Bbb A
\end{aligned}
$$

となる。
引数が常に`Unit`の唯一の値`()`固定ならばあとは`A`のどれかの要素を返すだけ、つまりただ`A`の要素から一つを選んでくるのに等しい。

#### 指数法則
関数表現での指数法則を確かめてみる。

まず$(B * C) ^ A = B ^ A * C ^ A$に対応するものを見てみると、

$$
\begin{aligned}
\Bbb A => Tuple2[B, C] &= \Bbb Tuple2[B, C] ^ \Bbb A \\
                        &= \Bbb Tuple2[B * C] ^ \Bbb A \\
                        &= \Bbb (\Bbb B * \Bbb C) ^ \Bbb A \\
                        &= \Bbb B ^ \Bbb A * \Bbb C ^ \Bbb A \\
                        &= \Bbb Tuple2[A => B, A => C]
\end{aligned}
$$

となる。
最初の形のほうが適用は簡単だが、たしかにタプルに入った別々の関数に各々`A`型の値を適用しても表現力としては同じである。

次に$C ^ {(A * B)} = (C ^ A) ^ B$に対応するものを見てみると、

$$
\begin{aligned}
\Bbb Tuple2[A, B] => C &= {\Bbb C} ^ {\Bbb Tuple2[A, B]} \\
            &= \Bbb C ^ {(\Bbb A * \Bbb B)} \\
            &= (\Bbb C ^ \Bbb A) ^ \Bbb B \\
            &= \Bbb B => A => C
\end{aligned}
$$

となる。
この事実は本当に面白くて、カリー化された関数とカリー化する前の関数が本質的には同じであるという事実が指数法則に対応することを示している。

### List
`List`がどういう計算体系に対応するかを見ていきたい。
まず`List`の定義は以下のよう。

```scala
sealed trait List[A]
case class Cons[A](head: A, next: List[A]) extends List[A]
case object Nil extends List[Nothing]
```

このデータ型の定式化はどうなるだろう？

まず`List`は`Cons`と`Nil`の直和なので以下が成り立つ。

$$
\Bbb List[A] = \Bbb Nil + \Bbb Cons[A]
$$

そして`Cons`は`head`と`next`の直積なので,

$$
\Bbb Nil + \Bbb Cons[A] = \Bbb Nil + \Bbb A * \Bbb List[A]
$$

が成り立つ。
更にこれらを繰り返し適用すると、

$$
\begin{aligned}
\Bbb List[A] &= \Bbb Nil + \Bbb Cons[A] \\
            &= \Bbb Nil + \Bbb A * \Bbb List[A] \\
            &= \Bbb Nil + \Bbb A * (\Bbb Nil + \Bbb Cons[A]) \\
            &= \Bbb Nil + \Bbb A * \Bbb Nil + \Bbb A * \Bbb Cons[A] \\
            &= \Bbb Nil + \Bbb A * \Bbb Nil + \Bbb A ^ 2 * \Bbb (\Bbb List[A]) \\
            &= \Bbb Nil + \Bbb A * \Bbb Nil + \Bbb A ^ 2 * \Bbb Nil + \Bbb A ^ 3 * \Bbb Nil ... \\
            &= 1 + \Bbb A + \Bbb A ^ 2 + \Bbb A ^ 3 + ... (\because \Bbb Nil = 1)
\end{aligned}
$$

のように無限級数になる。
何じゃこりゃと思えるかもしれないが、実際のデータ型に対応させてみるとわかりやすい。
`List`はそもそもどんなデータ構造だったかと言うと、任意の長さの`A`の要素を並べたものである。
例えば空っぽの`List`=`List()`はただ一通りしか存在しない。
長さが1の`List`は`A`のどの要素を一つ格納するかで$\Bbb A$通りある。
長さ2の場合は１つ目の格納する要素がどれかで$\Bbb A$通り、２つ目も同様に$\Bbb A$通りあるから全体として$\Bbb A ^ 2$
...
長さnの場合は同様に$\Bbb A ^ n$通り。
という感じになる。
つまり$1 + \Bbb A + \Bbb A ^ 2 + \Bbb A ^ 3 + ...$の各項は`List`の長さに応じたデータのバリエーションを表している。

さて


## 参考
- [The algebra (and calculus!) of algebraic data types](https://codewords.recurse.com/issues/three/algebra-and-calculus-of-algebraic-data-types)
- [Why do functional programmers always talk about algebra(s)?](https://arosien.github.io/fp-and-algebras/#1)
- [Zippers, Part 2: Zippers as Derivatives](https://pavpanchekha.com/blog/zippers/derivative.html#sec-2)
- [代数的データ型と初等代数学 - ryota-ka’s blog](https://ryota-ka.hatenablog.com/entry/2018/07/09/110000#f-f0ff00a3)
- [The Algebra of Algebraic Data Types](https://about.chatroulette.com/posts/algebraic-data-types/)





