# はじめに

Scalaのライブラリであるshapelessの使い方を追っているうちに、代数的データ型について調べる機会があったので、調べた内容についてまとめました。

# shapelessってどんなライブラリ?
　公式では以下のように紹介されています。
> shapeless is a type class and dependent type based generic programming library for Scala. 

　簡単に意訳すると、shapelessは型クラスと依存型を使って、Scalaでジェネリックプログラミングを実現するためのライブラリです。shapelessを使っているライブラリはsprayやspecs2、現在ではMagnoliaに変更されました[^1]が、doobieでもshapelessを使って実装されていました。  
　そんなshapelessですが、その機能の一つに代数的データ型と後述するHListを変換する機能があります。その機能を紹介する前に代数的データ型について紹介します。


# 代数的データ型(Algebratic Data Type; ADT)
　代数的データ型とは関数型プログラミングで使われている概念で、直積型と直和型を組み合わせて表現されるデータ型です。近年ではHaskellのような純粋関数型プログラミング言語だけではなく、ScalaやRust、Swiftでも使われています。

## 直積型(product type)
　「AはBとCから成る」のように複数の値の組み合わせを表現する時に使う型です。例えば、中学校1年生で習う座標は `(1, 2)` のように表しますが、これをイメージするとわかりやすいかもしれないです。この場合はFloat型のXとFloat型のYのペアで成り立つ直積型と考えられます。  
　Scalaはこの直積型をcase classやタプルを使って表します。例えばcase classを使う場合は
```scala
case class Coordinate(x: Float, y: Float)
```
のように書き、タプルを使う場合は
```scala
type Coordinate = (Float, Float)
```
のように書きます。
　
## 直和型(sum type; coproduct; variant)
　「AはBかCから成る」のように複数の型のどれかに当てはまるような型を直和型といいます。そして、直和型のそれぞれの型は直積型のように引数を取り作られます。  
　Scalaではこの直和型をtraitとそれを継承したclassを使って表す方法とEitherを使って表す方法があります。例えば、以下の仕様をScalaのコードで表すとこのようになります。

- 図形は円か長方形で成る
- 円は半径を取る
- 長方形は底辺と高さをとる

```scala
sealed trait Shape
final case class Circle(radius: Int) extends Shape
final case class Rectangle(width: Float, height: Float) extends Shape
```
もしくはEitherを使って表すと
```scala
type Circle = Int
type Rectangle = (Float, Float)
type Shape = Either[Rectangle, Circle] 
```
のように表します。  
　また、Scala3ではenumを使って以下のように直和型を書けるようになりました。
```scala
enum Shape:
    case Circle(radius: Int)
    case Rectangle(width: Float, height: Float)
```

## 列挙型は直和型なのか?
　直和型を見て、これは列挙型なのでは?と感じた方もいらっしゃるかもしれません。実際にScalaでも以下のように列挙型を表すことがあります。
```scala
sealed trait Color
final case class Red extends Color
final case class Blue extends Color
final case class Yellow extends Color
```
個人的にはあまり見たことがないですが、以下のように表すこともできるようです。
```scala
object Color extends Enumeration {
    val Red, Blue, Yellow = Value
}
```
　コードでの表現方法が似ているので、列挙型 = 直和型なのではという疑問が出てくるかもしれません。しかし、直和型の定義を説明する時に、「そして、直和型のそれぞれの型は直積型のように引数を取り作られます。」と説明しました。列挙型ではRed, Blue, Yellowに引数を渡すことができません。なので、これら列挙型は直和型とは言えません。この記事を書くときに参考にした[英語版のwikipedia](https://en.wikipedia.org/wiki/Algebraic_data_type)にも
> Enumerated types are a special case of sum types in which the constructors take no arguments, as exactly one value is defined for each constructor.

のように列挙型は直和型の特殊なケースと書かれています。

# shapelessを使って、代数的データ型とHListを行き来する
　さて、ここからが本題です。  
　shapelessは代数的データ型とHListを互いに変換する機能があります。HListとはHeterogenerous Listの略で、直訳すると異成分からなるリストと訳せます。このHListは異なる型を持つ値をひとまとめのリスト形式で表すことができる型です。
例えば、Int型とString型、Boolean型が組み合わさった型として以下の様に使うことができます。
```scala
val hoge: Int :: String :: Boolean :HNil = 1 :: "hoge":: true :: HNil
```
このHListは何かの型と何かの型が組み合わさった型と表現する様に先程の直積型と似た構造を持ちます。したがって、shapelessでは以下の様に直積型とHListを行き来することができます。
```scala
case class Hoge(foo: Int, bar: String, baz: Boolean)
val gen = Generic[Hoge]

val productHoge = Hoge(1, "fuga", true)
gen.to(productHoge)
//res: gen.Repr = 1 :: "fuga" :: true :: HNil

val hlistHoge = 2 :: "piyo" :: false :: HNil
gen.from(hlistHoge)
//res: Hoge = Hoge(2, "piyo", false)
```

# 現場での使用例
　この代数的データ型とHListを行き来する機能を使う場面としては例えば、以下の2点があります。
1. EntityとDTOの変換
2. 似たデータ構造を受け取り、特定のフォーマットに変換する関数の実装

## EntityとDTOの変換
　例えば、DBとアプリケーションとの間でデータのやりとりをする時にはEntityとDTO(data transfer object)を相互に変換してやりとりをする場面がしばしばあります。
```scala
case class HogeEntity(foo: Int, bar: String, baz: Boolean)
case class HogeDto(foo: Int, bar: String, baz: Boolean)
```
このEntityとDTOの間でデータを詰め替える時にshapelessが威力を発揮します。shapelessを使わずに実装すると
```scala
def toEntity(dto: HogeDto): HogeEntity = HogeEntity(dto.foo, dto.bar, dto.baz)
def toDto(entity: HogeEntity): HogeDto = HogeDto(entity.foo, entity.bar, dto.baz)
```
となるところが、shapelessを使うと
```scala
def getRepr[A](value: A)(implicit gen: Generic[A]) = gen.to(value)

def toEntity(dto: HogeDto): HogeEntity = Generic[HogeEntity] = gen.from(getRepr(dto))
def toDto(entity: HogeEntity): HogeDto = Generic[HogeDto] = gen.from(getRepr(entity))
```
と書くことができます。今回は3つの値しか持っていませんがこれが、10や20ものの値を持つクラスならばshapelessを使って、EntityとDtoを変換してあげると楽に変換することができます。(そもそも、巨大なクラスは適切なサイズに切り分けてあげるのがいいですが...)

## 似たデータ構造の値を受け取る関数の実装
こちらはUnderscoreの[The Type Astronaut’s Guide to Shapeless](https://books.underscore.io/shapeless-guide/shapeless-guide.html)でも出ている例の引用ですが、
```scala
case class Employee(name: String, number: Int, manager: Boolean)

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```
の様な似たデータ構造を持つクラスを一つの関数で受け取り、処理をする場合に使えます。
例えばこれらをCSVに変換する時にshapelessを使わない場合は
```scala
def employeeCsv(e: Employee): List[String] =
  List(e.name, e.number.toString, e.manager.toString)

def iceCreamCsv(c: IceCream): List[String] =
  List(c.name, c.numCherries.toString, c.inCone.toString)
```
の様に実装するのに対して、shapelessを使うと以下の様に実装することができます。
```scala
def genericCsv(gen: String :: Int :: Boolean :: HNil): List[String] =
  List(gen(0), gen(1).toString, gen(2).toString)

genericCsv(Generic[Employee].to(Employee("Dave", 123, false)))
genericCsv(Generic[IceCream].to(IceCream("Sundae", 1, false)))
```

# 参考文献
[Haskell wiki](https://wiki.haskell.org/Algebraic_data_type)  
[The Type Astronaut’s Guide to Shapeless](https://books.underscore.io/shapeless-guide/shapeless-guide.html)  
[Qiita「ADT, 直和・直積, State Machine」](https://qiita.com/ymtszw/items/dff02ad6350032688676)   
[wikipedia Algebraic data type](https://en.wikipedia.org/wiki/Algebraic_data_type)  


[^1]: 2021年1月9日の[こちらのコミット](https://github.com/tpolecat/doobie/commit/109ffe6ce5dfa032160435a163c8192474722267#diff-5634c415cd8c8504fdb973a3ed092300b43c4b8fc1e184f7249eb29a55511f91)でshapelessからMagnoliaにリプレイスされました。
