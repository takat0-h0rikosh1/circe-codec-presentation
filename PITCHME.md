@size[2.0em](JSON Codec を楽しもう<br>現場で役立つ circe)

+++

@snap[east]
![](t_horikoshi_profile.jpg) 
@snapend

@snap[west]
@size[0.8em](自己紹介)<br>
@size[0.8em](● 名前: 堀越 貴斗)<br>
@size[0.8em](● 会社: セプテーニ・オリジナル)<br>
@size[0.8em](● 職種: ソフトウェアエンジニア)<br>
@size[0.8em](● 趣味: 筋トレ, サウナ, ドライブ)<br>
@size[0.8em](● 好物: Scala, AWS, プロテイン)<br>
@size[0.8em](● Twitter:) [@tkt_hoorie](https://twitter.com/tkt_hoorie)<br>
@snapend

+++

#### はじめに

Webアプリケーションなんかを開発していると、<br>
例として Http Request/Response を処理するのに大抵は Json を扱いますよね。  

わたしは Scala を触り始めてから長らく play-json と歩みを共にしてきたのですが、<br>
最近(今更)、 circe を触ってみて大変便利でしたので実コードと解説を交えながら紹介していこうかと思います。

+++

#### アジェンダ
@ul
- Scala の JSON ライブラリ circe の使い方について解説
- [公式ドキュメント](https://circe.github.io/circe/)を題材に主要箇所を Pick up して説明
@ulend

+++

#### 準備

circe の依存関係を build.sbt に追加

```scala
val circeVersion = "0.10.0"

libraryDependencies ++= Seq(
  "io.circe" %% "circe-core",
  "io.circe" %% "circe-generic",
  "io.circe" %% "circe-parser"
).map(_ % circeVersion)
```

+++

@size[1.5em](Parsing)

+++

circe-parser に parse module が実装されている

@[1-11]
@[12-23]
@[25-27]
@[29]
@[30]
@[31-46]

```scala
scala> val jsonString: String = """{
     |   "foo": "foo value",
     |   "bar": {
     |      "bar_child": "bar child value"
     |    },
     |   "array":[
     |     { "content": 1 },
     |     { "content": 2 },
     |     { "content": 3 }
     |   ]
     | }"""
jsonString: String =
{
  "foo": "foo value",
  "bar": {
     "bar_child": "bar child value"
   },
  "array":[
    { "content": 1 },
    { "content": 2 },
    { "content": 3 }
  ]
}

scala> import io.circe._, io.circe.parser._
import io.circe._
import io.circe.parser._

scala> val parsed = parse(jsonString)
parsed: Either[io.circe.ParsingFailure,io.circe.Json] =
Right({
  "foo" : "foo value",
  "bar" : {
    "bar_child" : "bar child value"
  },
  "array" : [
    {
      "content" : 1
    },
    {
      "content" : 2
    },
    {
      "content" : 3
    }
  ]
})
```

+++

Parsing Failure

```scala
scala> val jsonString: String = "Invalid JSON String"
jsonString: String = Invalid JSON String

scala> val parsed = parse(jsonString)
parsed: Either[io.circe.ParsingFailure,io.circe.Json] =
  Left(io.circe.ParsingFailure: expected json value got I (line 1, column 1))
```

@[1-2]
@[4]
@[5-6]

+++

@size[1.5em](Traversing and modifying)

+++

Cursor という概念によって JSON を変換・抽出する

```scala
scala> val jsonString = """{
     |   "id": "c730433b-082c-4984-9d66-855c243266f0",
     |   "name": "Foo",
     |   "values": {
     |     "bar": true,
     |     "baz": 100.001,
     |     "qux": [
     |       "a",
     |       "b"
     |     ]
     |   }
     | }"""

scala> val doc: Json = parse(jsonString).getOrElse(Json.Null)
doc: io.circe.Json =
{
  "id" : "c730433b-082c-4984-9d66-855c243266f0",
  "name" : "Foo",
  "values" : {
    "bar" : true,
    "baz" : 100.001,
    "qux" : [
      "a",
      "b"
    ]
  }
}

scala> val cursor: HCursor = doc.hcursor
cursor: io.circe.HCursor = io.circe.cursor.TopCursor@7aca17af
```

@[1-12]
@[14-27]
@[29-30]

+++

@size[1.5em](Extracting data)

+++

HCursor の downField を使って抽出処理

```scala
// {
//    "id": "c730433b-082c-4984-9d66-855c243266f0",
//    "name": "Foo",
//    "values": {
//      "bar": true,
//      "baz": 100.001, // 取り出すよ!!!
//      "qux": [
//        "a",
//        "b"
//      ]
//    }
//  }

scala> val baz = cursor.downField("values").downField("baz").as[Double]
baz: io.circe.Decoder.Result[Double] = Right(100.001)

scala> val baz2 = cursor.downField("values").get[Double]("baz")
baz2: io.circe.Decoder.Result[Double] = Right(100.001)
```

@[1-12]
@[4-10]
@[6]
@[14-15]
@[17-18]

+++

配列の抽出

```scala
// {
//    "id": "c730433b-082c-4984-9d66-855c243266f0",
//    "name": "Foo",
//    "values": {
//      "bar": true,
//      "baz": 100.001, 
//      "qux": [
//        "a",
//        "b"
//      ] // この配列を取り出すよ!!!
//    }
//  }

scala> val qux = cursor.downField("values").downField("qux").as[Seq[String]]
qux: io.circe.Decoder.Result[Seq[String]] = Right(List(a, b))

scala> val secondQux = cursor.downField("values").downField("qux").downArray.right.as[String]
secondQux: io.circe.Decoder.Result[String] = Right(b)
```

@[1-12]
@[4-10]
@[7-10]
@[14-15]
@[17-18]

+++

@size[1.5em](Transforming data)

+++

フィールドに対する変換処理をサポート<br>
(文字列を逆さに)

```scala
// {
//    "id": "c730433b-082c-4984-9d66-855c243266f0",
//    "name": "Foo", // 文字列を逆さにするよ!!!
//    "values": {
//      "bar": true,
//      "baz": 100.001,
//      "qux": [
//        "a",
//        "b"
//      ]
//    }
//  }

scala> val reversedNameCursor: ACursor =
     |   cursor.downField("name").withFocus(_.mapString(_.reverse))
reversedNameCursor: io.circe.ACursor = io.circe.cursor.ObjectCursor@42127805

scala> val reversedNameJson = reversedNameCursor.top
reversedNameJson: Option[io.circe.Json] =
Some({
  "id" : "c730433b-082c-4984-9d66-855c243266f0",
  "name" : "ooF", // 逆さになったよ!!!
  "values" : {
    "bar" : true,
    "baz" : 100.001,
    "qux" : [
      "a",
      "b"
    ]
  }
})
```

@[1-12]
@[3]
@[14-16]
@[18]
@[19-31]

+++

@size[1.5em](Codec<br>Encode/Decode)

+++

Encode するためには @color[rebeccapurple](Encoder)<br>
Decode するためには @color[rebeccapurple](Decoder)<br>
それぞれ暗黙インスタンス(implicit)が必要

+++

- @size[1.0em](@color[rebeccapurple](Encoder[A]) は @color[rebeccapurple](A から Json) へ)
- @size[1.0em](@color[rebeccapurple](Decoder[A]) は @color[rebeccapurple](Json から Either[DecodingFailure, A]) へ)

+++

Scala 標準ライブラリの大半は circe がサポート

```scala
scala> import io.circe.syntax._, io.circe.parser.decode
import io.circe.syntax._
import io.circe.parser.decode

scala> val json = List(1, 2, 3).asJson
json: io.circe.Json =
[
  1,
  2,
  3
]

scala> val decodedFromJson = json.as[List[Int]]
decodedFromJson: io.circe.Decoder.Result[List[Int]] = Right(List(1, 2, 3))

scala> val decodedFromString = decode[List[Int]](json.noSpaces)
decodedFromString: Either[io.circe.Error,List[Int]] = Right(List(1, 2, 3))
```

+++

import io.circe.generic.auto._<br>
を宣言すれば case class も

```scala
scala> import io.circe.syntax._, io.circe.generic.auto._
import io.circe.syntax._
import io.circe.generic.auto._

case class Person(name: String)
case class Greeting(
   salutation: String,
   person: Person,
   exclamationMarks: Int
)

scala> val encoded = Greeting("Hey", Person("Chris"), 3).asJson
encoded: io.circe.Json =
{
  "salutation" : "Hey",
  "person" : {
    "name" : "Chris"
  },
  "exclamationMarks" : 3
}

scala> val decoded = encoded.as[Greeting]
decoded: io.circe.Decoder.Result[Greeting] = Right(Greeting(Hey,Person(Chris),3))
```
@size[0.5em](※ case class のフィールドはプリミティブ型など circe でサポートされている必要がある)

+++

@size[1.5em](Custome Codec)

+++

JSON のフィールド名を
明示的に指定したい

+++

Encoder, Decoder の定義に <br> forProductN を使うと良さそう

```scala
import io.circe.{ Decoder, Encoder }, io.circe.syntax._

case class User(id: Long, firstName: String, lastName: String)

implicit val decoderUser: Decoder[User] =
  Decoder.forProduct3("id", "first_name", "last_name")(User)

implicit val encoderUser: Encoder[User] =
  Encoder.forProduct3("id", "first_name", "last_name")(u =>
    (u.id, u.firstName, u.lastName))
```

+++

Encode 時にスネークケースに

```scala
scala> val userJson = User(123, "first name", "last name").asJson
userJson: io.circe.Json =
{
  "id" : 123,
  "first_name" : "first name",
  "last_name" : "last name"
}

scala> val decoded = userJson.as[User]
decoded: io.circe.Decoder.Result[User] = Right(User(123,first name,last name))
```

+++

Codec の実装を拡張したい

+++

Encoder, Decoder を独自に定義

```scala
import io.circe.{Decoder, Encoder, HCursor, Json}

class Thing(val foo: String, val bar: Int)

implicit val encoder: Encoder[Thing] =  new Encoder[Thing] {
  final def apply(t: Thing): Json = Json.obj(
    ("foo", Json.fromString(s"It's a ${t.foo}")),
    ("bar", Json.fromInt(t.bar * 1000))
  )
}

implicit val decoder: Decoder[Thing] = new Decoder[Thing] {
  final def apply(c: HCursor): Decoder.Result[Thing] =
    for {
      foo <- c.downField("foo").as[String]
      bar <- c.downField("bar").as[Int]
    } yield {
      new Thing(foo, bar)
    }
}
```

+++

Codec を思いのままに

```
scala> val thing = new Thing("test", 123)
thing: Thing = Thing@18fdb870

scala> val encoded = thing.asJson
encoded: io.circe.Json =
{
  "foo" : "It's a test",
  "bar" : 123000
}

scala> val decoded = encoded.as[Thing]
decoded: io.circe.Decoder.Result[Thing] = Right(Thing@66b66639)
```

+++

#### まとめ
- Parsing Module は circe に実装済
- 変換・抽出には Cursor を使う
- Scala標準クラスの Codec は circe がサポート
- Encoder/Decoder の定義で Codec を思いのままに

+++

@size[1.5em](Enjoy JSON Codec)

+++

@size[1.5em](ThanK you♡)

