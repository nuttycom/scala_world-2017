% Describing Data
% Kris Nuttycombe (@nuttycom)
% September, 2017

# Goals

* Explore the design of an FP library from start to finish
* Introduce **free applicative functors** 
* Use some **fix-point types** 
* Simplify with **recursion schemes** 
* Talk about the [**xenomorph**](https://github.com/nuttycom/xenomorph) library
* Discover how what we write at kind `* -> *` is related to what we write at kind `*`

# Prerequisites

* Sum types / GADTs
* Product types
* Applicative functors
* Higher-kinded types / type constructors
* Type-level lambdas / kind-projector
* Natural transformations
* Coproducts

## Resources
* Slides: [nuttycom.github.io/scala_world-2017](http://nuttycom.github.io/scala_world-2017)
* Sources: [github.com/nuttycom/scala_world-2017](https://github.com/nuttycom/scala_world-2017)
* Xenomorph: [github.com/nuttycom/xenomorph/tree/tutorial](https://github.com/nuttycom/xenomorph/tree/tutorial)

# Overview

## Problem: Serialization

* Having to maintain both serializers and deserializers is silly.
* Problems exist with macros/generic programming approaches.
* Legacy wire formats & evolving protocols can present challenges.

<div class="incremental"><div>
## Solution:

* Build a description of the data structure as a value. 
* Build interpreters for that description that produce serializers, 
  deserializers, and more.
</div></div>

<div class="notes">
This is a composition of talks, Rob Norris and John De Goes

What I'm presenting here are not really new ideas, just taking
some existing ideas and composing them in a slightly new way

The problem with generic programming approaches is that serialized
form becomes coupled to the type being represented, making it harder 
to change data structures. Your serialized form is your public API; it
needs to be stable and to have a controlled upgrade path.

Dissatisfaction-Driven Design
</div>

# Example

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Instant,
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
final case class Administrator(department: String, subordinateCount: Long) extends Role
~~~

<div class="incremental">
* Primitives
</div>
<div class="incremental">
* Sequences
</div>
<div class="incremental">
* Records
</div>
<div class="incremental">
* Sum types
</div>

# Example JSON Representation

~~~json
{
  "name": "Kris Nuttycombe",
  "birthDate": 201470280000,
  "roles": [
    {
      "admin": { 
        "department": "windmill-tilting",
        "subordinateCount": 0
      }
    } 
  ]
}
~~~

~~~json
{
  "name": "Jon Pretty",
  "birthDate": 411436800000,
  "roles": [
    {
      "user": {}
    }
  ]
}
~~~

# Example Schema - Product

~~~scala
val personSchema: Schema[Prim, Person] = rec(
  ^^(
    required("name",      Prim.str,                            Person.name.asGetter),
    required("birthDate", Prim.long composeIso isoLongInstant, Person.birthDate.asGetter),
    required("roles",     Prim.arr(roleSchema),                Person.roles.asGetter)
  )(Person.apply _)
)

val isoLongInstant = Iso(new Instant(_:Long))((_:Instant).getMillis)
~~~

To avoid type ascriptions, use all of [\@tpolecat](https://twitter.com/tpolecat)'s flags from [here](http://tpolecat.github.io/2017/04/25/scalac-flags.html)
 
# Example Schema - Sum
 
~~~scala
val roleSchema: Schema[Prim, Role] = Schema.oneOf(
  alt[Prim, Role, User.type](
    "user", 
    Schema.const(User),
    User.prism
  ) ::
  alt[Prim, Role, Administrator](
    "administrator", 
    rec(
      ^(
        required("department",       Prim.str, Administrator.department.asGetter),
        required("subordinateCount", Prim.int, Administrator.subordinateCount.asGetter)
      )(Administrator.apply _)
    ),
    Administrator.prism
  ) :: Nil
)
~~~

# Primitives

Use a GADT to describe the kinds of elements that can exist.

~~~scala
sealed trait JSchema[A]

case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
~~~

# Primitive Serialization

Use a GADT to describe the kinds of elements that can exist.

~~~scala
sealed trait JSchema[A]

case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
~~~

~~~scala
import argonaut.Json
import argonaut.Json._


def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    case JBoolT => jBool(value)
    case JStrT  => jString(value)
    case JNumT  => jNumber(value)
  }
}
 
~~~

<div class="notes">
We can write a serializer that can render any type for which we
have a schema to JSON.
</div>

# Primitive Parsing

Create a parser by generating a function between
a schema and an [argonaut](http://argonaut.io) DecodeJson instance.

~~~scala
sealed trait JSchema[A]

case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
~~~

~~~scala
import argonaut.DecodeJson
import argonaut.DecodeJson._


def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    case JBoolT => BooleanDecodeJson
    case JStrT  => StringDecodeJson
    case JNumT  => LongDecodeJson
  }
}
 
~~~

# Primitive Parsing

Create a parser by generating a function between
a schema and an [argonaut](http://argonaut.io) DecodeJson instance.

~~~scala
sealed trait JSchema[A]

case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
~~~

~~~scala
import argonaut.DecodeJson
import argonaut.DecodeJson._


  def apply[A](schema: JSchema[A]): DecodeJson[A] = {
    schema match {
      case JBoolT => BooleanDecodeJson
      case JStrT  => StringDecodeJson
      case JNumT  => LongDecodeJson
    }
  }
  
~~~

# Primitive Parsing

Create a parser by generating a ~~function~~ natural transformation between
a schema and a DecodeJson instance.

~~~scala
sealed trait JSchema[A]

case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
~~~

~~~scala
import argonaut.DecodeJson
import argonaut.DecodeJson._

val decoder = new (JSchema ~> DecodeJson) {
  def apply[A](schema: JSchema[A]): DecodeJson[A] = {
    schema match {
      case JBoolT => BooleanDecodeJson
      case JStrT  => StringDecodeJson
      case JNumT  => LongDecodeJson
    }
  }
}
~~~

# Natural Transformations

With a little rearranging, we can also make serialization a natural transformation.

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json
~~~

# Natural Transformations

With a little rearranging, we can also make serialization a natural transformation.

~~~scala
def serialize[A](schema: JSchema[A]): A => Json
~~~

<div class="incremental">
~~~scala
val serializer: JSchema ~> (? => Json)
~~~
</div>

<div class="incremental">
~~~scala
val serializer = new (JSchema ~> (? => Json)) {
  def apply[A](schema: JSchema[A]): A => Json = {
    schema match {
      case JBoolT => jBool(_)
      case JStrT  => jString(_)
      case JNumT  => jNumber(_)
    }
  }
}
~~~
</div>

# Natural Transformations

When we're working at the level of descriptions of data,
what we end up writing are natural transformations 
between our description and value-level functions.

~~~scala
val serializer: JSchema ~> (? => Json)
~~~

~~~scala
val decoder: JSchema ~> JsonDecoder
~~~

# Natural Transformations

When we're working at the level of descriptions of data,
what we end up writing are natural transformations 
between our description and value-level functions.

~~~scala
val serializer: JSchema ~> (? => Json)
~~~

~~~scala
val decoder: JSchema ~> (Json => Either[ParseError, ?])
~~~

# Sequences

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

<div class="incremental">
~~~scala
val boolsSchema: JSchema[Vector[Boolean]] = JVecT(JBoolT)
~~~
</div>

<div class="incremental">
~~~scala
val serializer = new (JSchema ~> (? => Json)) = {
  def apply[A](schema: JSchema[A]): A => Json = {
    schema match {
      case JVecT(elemSchema) => 
        value => jArray(value.map(serializer(elemSchema)))
      //...
    }
  }
}
~~~
</div>

# Sequence Parsing

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

~~~scala
val boolsSchema: JSchema[Vector[Boolean]] = JVecT(JBoolT)
~~~

~~~scala
val decoder = new (JSchema ~> DecodeJson) {
  def apply[A](schema: JSchema[A]) = {
    schema match {
      case JVecT(elemSchema) => 
        VectorDecodeJson(decoder(elemSchema))
      //...
    }
  }
}
~~~

# Records

For records, however, we have to be able to relate schema for
multiple values of different types.

~~~scala
case class Person(
  name: String, 
  birthDate: Instant
)
~~~

<div class="incremental">
~~~scala
def liftA2[A, B, C, F[_]: Applicative](fa: F[A], fb: F[B])(f: (A, B) => C): F[C]
~~~
</div>

<div class="incremental">
~~~scala
val personSchema: JSchema[Person] = liftA2(JStrT, JNumT) { Person.apply _ }
~~~
</div>

# Records

So, how do we define Applicative for this data type?

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]

case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

<div class="incremental">
~~~scala
trait Applicative[F[_]] {
  def pure[A](a: A): F[A]
  def map[A, B](fa: F[A])(f: A => B): F[B]
  def ap[A, B](fa: F[A])(ff: F[A => B]): F[B]
}
~~~

`pure` and `map` don't make any sense!
</div>

# Records, Take 2

We need an applicative functor, but maybe not for the whole schema type.
What about just for the record (product) types?

~~~scala
case class JObjT[A](props: Props[A]) extends JSchema[A]
~~~

<div class="incremental"><div>
To define our record builder first define a class that captures
the name and the schema for a single property. 

~~~scala
case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~
</div></div>

<div class="incremental"><div>
Now, wrap **that** up in a data structure for which we can define the applicative
operations.

~~~scala
sealed trait Props[O, A] 

case class ApProps[O, A, B](
  prop: PropSchema[O, B], 
  rest: Props[O, B => A]
) extends Props[O, A]

case class PureProps[O, A](a: A) extends Props[O, A]
~~~
</div></div>

<div class="notes">
This is also why we don't define Pure as a member of our base algebra. We don't need 
the other property information - the field name and the accessor - for every 
constructor.
</div>


# Records, Take 2

~~~scala
def applicative[O] = new Applicative[Props[O, ?]] {
  def point[A](a: => A): Props[O, A] = PureProps(a)

  override def map[A, B](fa: Props[O, A])(f: A => B): Props[O, B] = {
    fa match {
      case PureProps(a) => PureProps(f(a))
      case ApProps(prop, rest) => ApProps(prop, map(rest)(f compose _))
    }
  }

  def ap[A,B](fa: => Props[O, A])(ff: => Props[O, A => B]): Props[O, B] = {
    def flip[I, J, K](f: I => (J => K)) = (j: J) => (i: I) => f(i)(j)

    ff match {
      case PureProps(f) => map(fa)(f)

      case aprb: ApProps[O, (A => B), i] =>
        ApProps(aprb.hd, ap(fa)(map(aprb.tl)(flip[i, A, B])))
    }
  }
}
~~~

<div class="notes">
So, is this thing applicative? Well, the easiest way to find out is to go ahead and
see if you can just implement the Applicative typeclass in a way that makes sense.
</div>

# Records, Take 2
~~~scala
sealed trait Props[O, A] 

case class PureProps[O, A](a: A) extends Props[O, A]

case class ApProps[O, A, B](
  prop: PropSchema[O, B], 
  rest: Props[O, B => A]
) extends Props[O, A]
~~~

~~~scala
case class JObjT[A](props: Props[A, A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

<div class="notes">
Now, this would work fine. But there's something to notice about Props - we could
add a bit of abstraction here by making the PropSchema constructor a type variable 
instead.
</div>

# Records, Take 2

~~~scala
sealed trait Props[F[_, _], O, A] 

case class PureProps[F[_, _], O, A](a: A) extends Props[F, O, A]

case class ApProps[F[_, _], O, A, B](
  prop: F[O, B], 
  rest: Props[F, O, B => A]
) extends Props[F, O, A]
~~~

~~~scala
case class JObjT[A](props: Props[PropSchema, A, A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

<div class="notes">
That `O` looks kind of superfluous.
</div>

# Records, Take 2

~~~scala
sealed trait Props[F[_], A] 

case class PureProps[F[_], A](a: A) extends Props[F, A]

case class ApProps[F[_], A, B](
  prop: F[B], 
  rest: Props[F, B => A]
) extends Props[F, A]
~~~

~~~scala
case class JObjT[A](props: Props[PropSchema[A, ?], A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

# Records, Take 3

~~~scala
sealed trait FreeAp[F[_], A] 

case class Pure[F[_], A](a: A) extends FreeAp[F, A]

case class Ap[F[_], A, B](
  head: F[B], 
  tail: FreeAp[F, B => A]
) extends FreeAp[F, A]
~~~

~~~scala
case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

# Records, Take 3

~~~scala
import scalaz.FreeAp

case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

# Record serialization

~~~scala
  def serializeObj[A](rb: FreeAp[PropSchema[A, ?], A], value: A): Json = {
    jObject(
      rb.foldMap[State[JsonObject, ?]](
        new (PropSchema[A, ?] ~> State[JsonObject, ?]) {
          def apply[B](ps: PropSchema[A, B]): State[JsonObject, B] = {
            val elem: B = ps.accessor(value)
            for {
              _ <- modify((_: JsonObject) + (ps.fieldName, serializer(ps.valueSchema)(elem)))
            } yield elem
          }
        }
      ).exec(JsonObject.empty)
    )
  }
~~~

<div class="incremental">
~~~scala
val serializer = new (JSchema ~> (? => Json)) = {
  def apply[A](schema: JSchema[A]): A => Json = {
    schema match {
      case JObjT(fa) => serializeObj(fa, _: A)
      //...
    }
  }
}
~~~
</div>

# Record decoding

~~~scala
  def decodeObj[A](rb: FreeAp[PropSchema[A, ?], A]): DecodeJson[A] = {
    implicit val djap: Applicative[DecodeJson] = new Applicative[DecodeJson] {
      def point[A](a: => A) = DecodeJson(_ => DecodeResult.ok(a))
      def ap[A, B](fa: => DecodeJson[A])(ff: => DecodeJson[A => B]): DecodeJson[B] = {
        fa.flatMap(a => ff.map(_(a)))
      }
    }

    rb.foldMap(
      new (PropSchema[A, ?] ~> DecodeJson) {
        def apply[B](ps: PropSchema[A, B]): DecodeJson[B] = DecodeJson(
          _.downField(ps.fieldName).as(decoder(ps.valueSchema))
        )
      }
    )
  }
~~~

<div class="incremental">
~~~scala
val decoder = new (JSchema ~> DecodeJson) {
  def apply[A](schema: JSchema[A]) = {
    schema match {
      case JObjT(rb) => decodeObj(rb, _: A)
      //...
    }
  }
}
~~~
</div>

# Sum types
We represent sum types as a list of alternatives.

Each constructor of the sum type is associated with a value that
maps from the arguments demanded by that constructor to a value
of the sum type. 

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])
~~~

<div class="incremental">
The combination of 'review' and 'preview' is better expressed as a prism.

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], prism: Prism[A, B])
~~~
</div>

# Sum types
~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], prism: Prism[A, B])
~~~

<div class="incremental">
~~~scala
import monocle.macros._

case object User extends Role {
  val prism = GenPrism[Role, User.type]
}

// "user": {}

val userRoleAlt = Alt[Role, Unit](
  "user", 
  JObjT(FreeAp.pure(())), 
  User.prism composeIso GenIso.unit[User.type]
)
~~~
</div>

# Sum types

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], prism: Prism[A, B])
~~~

~~~scala
import monocle.macros._

final case class Administrator(department: String, subordinateCount: Long) extends Role
object Administrator {
  val prism = GenPrism[Role, Administrator]
}

// { "admin": { "department": "windmill-tilting", "subordinateCount": 0 } } 

val adminRoleAlt = Alt[Role, Administrator](
  "admin", 
  JObjT(
    ^(
      FreeAp.lift(PropSchema("department", JStrT, (_:Administrator).department)),
      FreeAp.lift(PropSchema("subordinateCount", JNumT, (_:Administrator).subordinateCount)),
    )(Administrator.apply _)
  ),
  Administrator.prism 
)
~~~

# Sum types

~~~scala
val roleSchema: JSchema[Role] = JSumT(userRoleAlt :: adminRoleAlt :: Nil)
~~~

# Sum type serialization

~~~scala
val serializer = new (JSchema ~> (? => Json)) = {
  def apply[A](schema: JSchema[A]): A => Json = {
    schema match {
      case JSumT(alts) => 
        (value: A) => alts.flatMap({
          case Alt(id, base, prism) => 
            prism.getOption(value).map(serializer(base)).toList map { json =>
              jObject(JsonObject.single(id, json))
            }
        }).head

      //...
    }
}
~~~

<div class="notes">
The partiality of results.head is actually not as bad as it looks; the reason 
that it's *arguably* excusable is that if you encounter a case where no
alternative is able to satisfy the serialization of that value, your 
program *should* just explode. In an ideal world we'd be able to statically
check the inhabitants of that array of alternatives against the constructors
of the algebraic data type that we're building a schema for, but unfortunately
the set of constructors is not a first-class value in any language that I 
know of.
</div>

# Sum type parsing

~~~scala
val decoder = new (JSchema ~> DecodeJson) {
  def apply[A](schema: JSchema[A]) = {
    schema match {
      //...
      case JSumT(alts) => DecodeJson { (c: HCursor) => 
        val results = for {
          fields <- c.fields.toList
          altResult <- alts flatMap {
            case Alt(id, base, prism) =>
              fields.exists(_ == id).option(
                c.downField(id).as(decoder(base)).map(prism.reverseGet)
              ).toList
          }
        } yield altResult 

        val altIds = alts.map(_.id)
        results match {
          case x :: Nil => x
          case Nil => DecodeResult.fail(s"No fields found matching any of ${altIds}", c.history)
          case xs => DecodeResult.fail(s"More than one matching field found among ${altIds}", c.history)
        }
      }
    }
  }
}
~~~

# What else can we do?

* ScalaCheck [Gen](https://github.com/rickynils/scalacheck/blob/master/src/main/scala/org/scalacheck/Gen.scala) instances
* Binary serialization using [scodec](https://github.com/scodec/scodec)
* [User interfaces](https://app-dev.pellucid.com)
* Interpret to whatever Applicative you want, really.

<div class="notes">
Franco Ponticelli, a coworker of mine, wrote a schema interpreter that
he uses to serialize values to database tables... and provides the 'CREATE TABLE'
statements accordingly.
</div>

# So... what's the catch?

<div class="incremental">
* Fixed set of primitives is overly limiting
</div>
<div class="incremental">
* Unable to provide additional metadata about values/fields 
</div>

# Problem 1: Primitives

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]


case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]

case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]
~~~

# Problem 1: Primitives

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
case object JDateT extends JSchema[DateTime]

case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]

case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]
~~~

# Problem 1: Primitives

~~~scala
sealed trait JSchema[P[_], A]

case class JPrimT[P[_], A](prim: P[A]) extends JSchema[P, A]




case class JVecT[P[_], A](elemType: JSchema[P, A]) extends JSchema[P, Vector[A]]

case class JSumT[P[_], A](alternatives: List[Alt[P, A, B] forSome { type B }]) extends JSchema[P, A]

case class JObjT[P[_], A](props: FreeAp[PropSchema[P, A, ?], A]) extends JSchema[P, A]
~~~

<div class="incremental">
~~~scala
sealed trait JsonPrim[A]
case object JStrT extends JsonPrim[String]
case object JNumT extends JsonPrim[Long]
case object JBoolT extends JsonPrim[Boolean]
case object JDateT extends JsonPrim[DateTime]
~~~
</div>

<div class="incremental">
~~~scala
type MySchema[A] = JSchema[JsonPrim, A]
~~~
</div>

# Problem 1: Primitives

~~~scala
trait ToJson[S[_]] {
  def serializer: S ~> (? => Json)
}

trait FromJson[S[_]] {
  def decoder: S ~> DecodeJson
}
~~~

# Problem 1: Primitives

~~~scala
implicit def jSchemaToJson[P[_]: ToJson] = new ToJson[JSchema[P, ?]] {
  val serializer = new (JSchema[P, ?] ~> (? => Json)) {
    def apply[A](schema: JSchema[P, A]): A => Json = {
      schema match {
        case JPrimT(p) => implicitly[ToJson[P]].serializer(p)

        // handling the other constructors stays the same
      }
    }
  }
}
~~~

<div class="incremental">
~~~scala
implicit val JsonPrimToJson = new ToJson[JsonPrim] {
  val serializer = new (JsonPrim ~> (? => Json)) {
    def apply[A](p: JsonPrim[A]): A => Json = {
      schema match {
        case JStrT  => jString(_)
        case JNumT  => jNumber(_)
        case JBoolT => jBool(_)
        case JDateT => (dt: DateTime) => jString(dt.toString)
      }
    }
  }
}
~~~
</div>

# Coproducts

~~~scala
implicit def primCoToJson[P[_]: ToJson, Q[_]: ToJson] = new ToJson[Coproduct[P, Q, ?]] {
  val serializer = new (Coproduct[P, Q, ?] ~> (A => Json)) {
    def apply[A](p: Coproduct[P, Q, A]): A => Json = {
      p.run.fold(
        implicitly[ToJson[P]].serializer,
        implicitly[ToJson[Q]].serializer
      )
    }
  }
}

implicit def primCoFromJson[P[_]: FromJson, Q[_]: FromJson] = new FromJson[Coproduct[P, Q, ?]] {
  val decoder = new (Coproduct[P, Q, ?] ~> DecodeJson) {
    def apply[A](p: Coproduct[P, Q, A]): DecodeJson[A] = {
      p.run.fold(
        implicitly[FromJson[P]].decoder,
        implicitly[FromJson[Q]].decoder
      )
    }
  }
}
~~~

# Coproducts

~~~scala
sealed trait MyPrim[A]
case object MyInt extends MyPrim[Int]
case object MyStr extends MyPrim[String]

implicit object MyPrimToJson extends ToJson[MyPrim] {
  val serializer = new (MyPrim ~> (? => Json)) {
    def apply[A](p: MyPrim[A]) = p match {
      case MyInt => jNumber(_)
      case MyStr => jString(_)
    }
  }
}
~~~

<div class="incremental">
~~~scala
sealed trait YourPrim[A]
case object YourInt extends YourPrim[Int]
case object YourDouble extends YourPrim[Double]

implicit object YourPrimToJson extends ToJson[YourPrim] {
  val serializer = new (YourPrim ~> (? => Json)) {
    def apply[A](p: YourPrim[A]) = p match {
      case YourInt => (i: Int) => jString(s"i${i}")
      case YourDouble => (d: Double) => jString(s"d${d}")
    }
  }
}
~~~
</div>

# Coproducts

~~~scala
type Both[A] = Coproduct[MyPrim, YourPrim, A]
def int: Both[Int] = rightc(YourInt)
def double: Both[Double] = rightc(YourDouble)
def str: Both[String] = leftc(MyStr)

implicitly[ToJson[Both]]
~~~

# Either and Coproduct

~~~scala
Either[A, B]
~~~

Allows you to take two sum types and create a new
sum type whose inhabitants are the union of the inhabitants of `A`
and the inhabitants of `B`.

<div class="incremental"><div>
~~~scala
Coproduct[F[_], G[_]]
~~~
Allows you to take two **descriptions** of (sets of) types and create a new
**description** which can describe values using either of the nested
descriptions.  
</div></div>

# Problem 2: Annotations

* Nodes of the JSchema tree don't currently contain enough
  information to generate interesting json-schema.
    * titles
    * min/max length of arrays
    * property order
    * formatting metadata
    * etc.
* We might want different kinds of metadata for different
  applications.

# Problem 2: Annotations

~~~scala
sealed trait JSchema[P[_], I]

case class JPrimT[P[_], I](prim: P[I]) extends JSchema[P, I]

case class JVecT[P[_], I](elemType: JSchema[P, I]) extends JSchema[P, Vector[I]]

case class JSumT[P[_], I](alternatives: List[Alt[P, I, J] forSome { type J }]) extends JSchema[P, I]

case class JObjT[P[_], I](props: FreeAp[PropSchema[P, I, ?], I]) extends JSchema[P, I]
~~~

# Problem 2: Annotations

~~~scala
sealed trait JSchema[P[_], A, I]

case class JPrimT[P[_], A, I](ann: A, prim: P[I]) extends JSchema[P, A, I]

case class JVecT[P[_], A, I](ann: A, elemType: JSchema[P, A, I]) extends JSchema[A, P, Vector[I]]

case class JSumT[P[_], A, I](ann: A, alternatives: List[Alt[P, A, I, J] forSome { type J }]) extends JSchema[P, A, I]

case class JObjT[A, P[_], I](ann: A, props: FreeAp[PropSchema[P, A, I, ?], I]) extends JSchema[P, A, I]
~~~

This is a bit messy and involves a bunch of redundancy. It will work; it just doesn't seem ideal.

# Directly recusive data

~~~scala
class Prof(
  name: String,
  students: List[Prof]
)
~~~

# No longer directly recusive data

~~~scala
class ProfF[S](
  name: String,
  students: List[S]
)
~~~

# Reintroducing recursion with Fix

~~~scala
class ProfF[S](
  name: String,
  students: List[S]
)
~~~

~~~scala
case class Fix[F[_]](f: F[Fix[F]])

type Prof = Fix[ProfF]
~~~

# Annotating a tree with Cofree

~~~scala
class ProfF[S](
  name: String,
  students: List[S]
)
~~~

~~~scala
case class Fix[F[_]](f: F[Fix[F]])

type Prof = Fix[ProfF]
~~~

~~~scala
case class Cofree[F[_], A](f: F[Cofree[F, A]], a: A)

type IdProf = Cofree[ProfF, Int]
~~~

Hat tip to Rob Norris, go watch his talk [here](https://www.youtube.com/watch?v=7xSfLPD6tiQ)

# Annotating a tree with Cofree

~~~scala
sealed trait Schema[P[_], I]

case class VecT[P[_], I](elemType: Schema[P, I]) extends Schema[P, Vector[I]]
~~~

<div class="incremental">
~~~scala
sealed trait SchemaF[P[_], S, I]

case class VecTF[P[_], S,  I](elemType: S) extends SchemaF[P, S, Vector[I]]
~~~

~~~scala
type Schema[P[_], I] = Fix[SchemaF[P, ?, I]]
~~~

Does this work?
</div>

<div class="notes">
This obviously won't work - we've lost the witness for the element type
of our vector. 
</div>

# Annotating a tree with ~~Cofree~~ HCofree

~~~scala
sealed trait SchemaF[P[_], F[_], I]

case class VecTF[P[_], F[_], I](elemType: F[I]) extends SchemaF[P, F, Vector[I]]
~~~

<div class="incremental">
~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])
~~~
</div>

<div class="incremental">
~~~scala
type Schema[P[_]] = HFix[SchemaF[P, ?[_], ?]]
~~~
</div>

<div class="incremental"><div>
To add annotations, use HCofree instead of HFix

~~~scala
case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~
</div></div> 

<div class="incremental">
~~~scala
type AnnSchema[P[_], A] = HCofree[A, SchemaF[P, ?[_], ?]]
~~~
</div>

# Rewriting our interpreters

~~~scala
type Schema[P[_]] = HFix[SchemaF[P, ?[_], ?]]
~~~

~~~scala
implicit def schemaToJson[P[_]: ToJson] = new ToJson[Schema[P, ?]] {
  val serializer = new (Schema[P, ?] ~> (? => Json)) = {
    def apply[I](schema: Schema[P, I]): I => Json = {
      schema.unfix match {
        case SumT(alts) => 
          (value: I) => alts.flatMap({ 
            case Alt(id, base, prism) => 
              prism.getOption(value).map(serializer(base)).toList map { 
                jObject(JsonObject.single(id, _))
              }
          }).head

        // ...
      }
    }
  }
}
~~~

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

<div class="incremental">
~~~scala
def forget[A, F[_[_], _], I](c: HCofree[A, F, I]): HFix[F, I]
~~~
</div>

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

~~~scala
def forget[A, F[_[_], _]]: (HCofree[A, F, ?] ~> HFix[F, ?])
~~~

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

~~~scala
def forget[A, F[_[_], _]]: (HCofree[A, F, ?] ~> HFix[F, ?]) = 
  new (HCofree[A, F, ?] ~> HFix[F, ?]) {
    def apply[I](c: HCofree[A, F, I]): HFix[F, I] = {
      c.f // this only discards one layer of annotation!
    }
  }
~~~

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

~~~scala
def forget[A, F[_[_], _]: HFunctor]: (HCofree[A, F, ?] ~> HFix[F, ?]) = 
  new (HCofree[A, F, ?] ~> HFix[F, ?]) { self => 
    def apply[I](c: HCofree[A, F, I]): HFix[F, I] = {
      HFix(implicitly[HFunctor[F]].hfmap(self)(c.f))
    }
  }
}
~~~

<div class="incremental">
~~~scala
trait HFunctor[F[_[_], _]] {
  def hfmap[M[_], N[_]](nt: M ~> N): F[M, ?] ~> F[N, ?]
}
~~~
</div>

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

<div class="incremental">
~~~scala
final case class HEnvT[A, F[_[_], _], G[_], I](a: A, fg: F[G, I])
~~~
</div>

# HFix and HCofree

~~~scala
case class HFix[F[_[_], _], I](unfix: F[HFix[F, ?], I])

//case class HCofree[A, F[_[_], _], I](a: A, f: F[HCofree[A, F, ?], I])
~~~

~~~scala
final case class HEnvT[A, F[_[_], _], G[_], I](a: A, fg: F[G, I])
~~~

~~~scala
type HCofree[A, F[_[_], _], I] = HFix[HEnvT[A, F, ?[_], ?], I]
~~~

# Working with fixpoint trees

~~~scala
type HAlgebra[F[_[_], _], G[_]] = F[G, ?] ~> G
~~~

~~~scala
def cata[F[_[_], _]: HFunctor, G[_]](alg: HAlgebra[F, G]): (HFix[F, ?] ~> G) = 
  new (HFix[F, ?] ~> G) { self => 
    def apply[I](f: HFix[F, I]): G[I] = {
      alg.apply[I](f.unfix.hfmap[G](self))
    }
  }
~~~

<div class="incremental">
~~~scala
//final case class HEnvT[A, F[_[_], _], G[_], I](a: A, fg: F[G, I])
//type HCofree[A, F[_[_], _], I] = HFix[HEnvT[A, F, ?[_], ?], I]

def forgetAlg[A, F[_[_], _]] = new HAlgebra[HEnvT[A, F, ?[_], ?], HFix[F, ?]] {
  def apply[I](env: HEnvT[A, F, HFix[F, ?], I]) = HFix(env.fg)
}
~~~
</div>

<div class="incremental">
~~~scala
def forget[F[_[_], _]: HFunctor, A]: (HCofree[A, F, ?] ~> HFix[F, ?]) = cata(forgetAlg)
~~~
</div>

# Drawbacks

The Scala compiler hates me.

~~~
[error] /Users/nuttycom/personal/scala_world-2017/sample_code/xenomorph/src/main/scala/xenomorph/Schema.scala:263: type mismatch;
[error]  found   : [γ$13$]xenomorph.PropSchema[O,[γ$40$]xenomorph
.HCofree[[β$0$[_$1], γ$1$]xenomorph
.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$13$] ~> [γ$14$]xenomorph.PropSchema[N,[γ$40$]xenomorph
.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$14$]
[error]     (which expands to)  scalaz.NaturalTransformation[[γ$13$]xenomorph
.PropSchema[O,[γ$40$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph
.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$13$],[γ$14$]xenomorph.PropSchema[N,[γ$40$]xenomorph
.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$14$]]
[error]  required: [γ$3$]xenomorph.PropSchema[O,[γ$2$]xenomorph
.HCofree[[β$0$[_$1], γ$1$]xenomorph
.SchemaF[P,β$0$,γ$1$],A,γ$2$],γ$3$] ~> G
[error]     (which expands to)  scalaz.NaturalTransformation[[γ$3$]xenomorph
.PropSchema[O,[γ$2$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph
.SchemaF[P,β$0$,γ$1$],A,γ$2$],γ$3$],G]
[error]       PropSchema.contraNT[O, N, Schema[A, P, ?]](f)
~~~

That's the result of leaving off a type ascription, there's actually nothing incorrect about the code.

# Conclusion

When we're working at kind `*`, we're dealing with data.

When we're working at kind `* -> *` we're dealing with **descriptions** of data.

<div class="incremental">
~~~scala
A => B

F[_] ~> G[_]
~~~
</div>

<div class="incremental">
~~~scala
Either[A, B]

Coproduct[F[_], G[_]]
~~~
</div>

<div class="incremental">
~~~scala
trait Functor[F[_]] {
  def fmap[A, B](f: A => B): F[A] => F[B]
}

trait HFunctor[F[_[_], _]] {
  def hfmap[M[_], N[_]](nt: M ~> N): F[M, ?] ~> F[N, ?]
}
~~~
</div>
