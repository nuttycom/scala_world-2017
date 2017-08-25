% Describing Data
% Kris Nuttycombe (@nuttycom)
% September, 2017

Goals
-----

* Introduce **free applicative functors** in the context of a "real-world" problem
* Discuss how **fix-point types** allow us to annotate recursive data structures
* See how these things come together in the **schematic** library
* Explore the design of an FP library from start to finish

Overview
--------

* This talk is about the design of a library using pure FP style.

* Dissatisfaction-Driven Design

* Problem: Serialization
    * Having to maintain both serializers and deserializers is silly
    * Problems exist with generic programming approaches 
    * Legacy wire formats can present challenges.

* Solution:
    * Build a description of the data structure that can be interpreted
      to derive serializers, deserializers, and more.

<div class="notes">
This is a composition of talks, Rob Norris and John De Goes

What I'm presenting here are not really new ideas, just taking
some existing ideas and composing them in a slightly new way

Functional programming is programming with values.

The problem with generic programming approaches is that serialized
form becomes coupled to the type being represented, making it harder 
to change data structures. Your serialized form is your public API; it
needs to be stable and to have a controlled upgrade path.
</div>

Outline
-------

* Build a library to solve the problem of JSON serialization
* Look at what else we might use it for
* Figure out how it's deficient
* Generalize it with fancy types

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
case class Administrator(department: String) extends Role
~~~

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
case class Administrator(department: String) extends Role
~~~

* Primitives

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
case class Administrator(department: String) extends Role
~~~

* Primitives
* Sequences

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
case class Administrator(department: String) extends Role
~~~

* Primitives
* Sequences
* Sum types

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
  roles: Vector[Role]
)

sealed trait Role

case object User extends Role
case class Administrator(department: String) extends Role
~~~

* Primitives
* Sequences
* Sum types
* Records

JSON Representation
-------------------

~~~json
{
  "name": "Kris Nuttycombe",
  "birthDate": 201470280000,
  "roles": [
    {
      "admin": { "department": "windmill-tilting" }
    } 
  ]
}

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

Primitives
----------

Use a GADT to describe the kinds of elements that can exist.

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Double]
case object JBoolT extends JSchema[Boolean]
~~~

Primitive serialization
-----------------------

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

Primitive parsing
-----------------

Create a parser by generating a natural transformation between
a schema and an [argonaut](http://argonaut.io) DecodeJson instance.

~~~scala
import argonaut.DecodeJson
import argonaut.DecodeJson._

def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    case JBoolT => BooleanDecodeJson
    case JStrT  => StringDecodeJson
    case JNumT  => DoubleDecodeJson
  }
}
~~~

Sequences
---------

Sequences are simple to describe because the only thing you need to represent
is the type of the element. Length information is gravy.

~~~scala

case class JVecT[A](elemType: JSchema[A], minLength: Int, maxLength: Option[Int]) extends JSchema[Vector[A]]

// EXAMPLE

val boolsSchema: JSchema[Vector[Boolean]] = JArrayT(JBoolT, 0, None)
~~~

Sequence serialization
-----------------------

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    //...
    case JVecT(elemSchema, _, _) => jArray(value.map(serialize(elemSchema, _)).toList)
  }
}
~~~

Sequence parsing
-----------------

~~~scala
def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    //...
    case JVecT(elemSchema, _, _) => VectorDecodeJson(decoder(elemSchema))
  }
}
~~~

Records
-------

~~~scala

case class Person(
  name: String, 
  birthDate: Double // seconds since the epoch
)

~~~

~~~scala

def liftA2[A, B, C, F[_]: Applicative](fa: F[A], fb: F[B])(f: (A, B) => C): F[C]

val personSchema: JSchema[Person] = liftA2(JStrT, JNumT) { Person.apply _ }

~~~

This looks like exactly the sort of thing that we need, but we immediately run
into a problem if we try to write Applicative[JSchema]. We can't even make JSchema
a functor!

Records, Take 2
---------------

~~~scala
case class JObjT[A](recordBuilder: RecordBuilder[A]) extends JSchema[A]
~~~

Records, Take 2
---------------

~~~scala
case class JObjT[A](recordBuilder: RecordBuilder[A, A]) extends JSchema[A]

sealed trait RecordBuilder[O, A] 

case class ApRecordBuilder[O, A, B](
  hd: PropSchema[O, B], 
  tl: RecordBuilder[O, B => A]
) extends RecordBuilder[O, A]

case class PureRecordBuilder[O, A](a: A) extends RecordBuilder[O, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 2
---------------

~~~scala
def applicative[O] = new Applicative[RecordBuilder[O, ?]] {
  def point[A](a: => A): RecordBuilder[O, A] = PureRecordBuilder(a)

  override def map[A,B](fa: RecordBuilder[O, A])(f: A => B): RecordBuilder[O, B] = {
    fa match {
      case PureRecordBuilder(a) => PureRecordBuilder(f(a))
      case ApRecordBuilder(hd, tl) => ApRecordBuilder(hd, map(tl)(f compose _))
    }
  }

  def ap[A,B](fa: => RecordBuilder[O, A])(ff: => RecordBuilder[O, A => B]): RecordBuilder[O, B] = {
    ff match {
      case PureRecordBuilder(f) => map(fa)(f)
      case aprb: ApRecordBuilder[O, (A => B), i] => 
        ApRecordBuilder(
          aprb.hd, 
          ap(fa) { 
            map[i => (A => B), A => (i => B)](aprb.tl) { 
              (g: i => (A => B)) => { (a: A) => { (i: i) => g(i)(a) } } // this is just flip
            }
          }
        )
    }
  }
}
~~~

Records, Take 2
---------------

~~~scala
case class JObjT[A](recordBuilder: RecordBuilder1[PropSchema, A, A]) extends JSchema[A]

sealed trait RecordBuilder1[F[_, _], O, A] 

case class ApRecordBuilder[F[_, _], O, A, B](
  hd: F[O, B], 
  tl: RecordBuilder1[F, O, B => A]
) extends RecordBuilder1[F, O, A]

case class PureRecordBuilder[F[_, _], O, A](a: A) extends RecordBuilder1[F, O, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 2
---------------

~~~scala
case class JObjT[A](recordBuilder: RecordBuilder1[({ type l[a] = PropSchema[A, a] })#l, A]) extends JSchema[A]

sealed trait RecordBuilder1[F[_], A] 

case class ApRecordBuilder[F[_], A, B](
  hd: F[B], 
  tl: RecordBuilder1[F, B => A]
) extends RecordBuilder1[F, A]

case class PureRecordBuilder[F[_], A](a: A) extends RecordBuilder1[F, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 3
---------------

~~~scala
case class JObjT[A](recordBuilder: FreeAp[({ type l[a] = PropSchema[A, a] })#l, A]) extends JSchema[A]

sealed trait FreeAp[F[_], A] 

case class Ap[F[_], A, B](
  hd: F[B], 
  tl: FreeAp[F, B => A]
) extends FreeAp[F, A]

case class Pure[F[_], A](a: A) extends FreeAp[F, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 3
---------------

~~~scala
import scalaz.FreeAp

case class JObjT[A](recordBuilder: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Record serialization
--------------------

~~~scala
  def serializeObj[A](rb: FreeAp[PropSchema[A, ?], A], value: A): Json = {
    jObject(
      rb.foldMap[State[JsonObject, ?]](
        new NaturalTransformation[PropSchema[A, ?], State[JsonObject, ?]] {
          def apply[B](ps: PropSchema[A, B]): State[JsonObject, B] = {
            val elem: B = ps.accessor(value)
            for {
              obj <- get
              _ <- put(obj + (ps.fieldName, serialize(ps.valueSchema, elem)))
            } yield elem
          }
        }
      ).exec(JsonObject.empty)
    )
  }
~~~

Record decoding
---------------

~~~scala
  def decodeObj[A](rb: FreeAp[PropSchema[A, ?], A]): DecodeJson[A] = {
    implicit val djap: Applicative[DecodeJson] = new Applicative[DecodeJson] {
      def point[A](a: => A) = DecodeJson(_ => DecodeResult.ok(a))
      def ap[A, B](fa: => DecodeJson[A])(ff: => DecodeJson[A => B]): DecodeJson[B] = {
        fa.flatMap(a => ff.map(_(a)))
      }
    }

    rb.foldMap(
      new NaturalTransformation[PropSchema[A, ?], DecodeJson] {
        def apply[B](ps: PropSchema[A, B]): DecodeJson[B] = {
          DecodeJson(_.downField(ps.fieldName).as(decoder(ps.valueSchema)))
        }
      }
    )
  }
~~~

Sum types
----

We represent sum types as a list of alternatives.

Each constructor of the sum type is associated with a value that
maps from the arguments demanded by that constructor to a value
of the sum type. 

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])
~~~

Sum types
----

We represent sum types as a list of alternatives.

Each constructor of the sum type is associated with a value that
maps from the arguments demanded by that constructor to a value
of the sum type. 

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])
~~~

Sum types
----

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])

// EXAMPLE
//
// case class Administrator(department: String) extends Role
// 
// {
//   "admin": { "department": "windmill-tilting" }
// } 
//
~~~

Sum types
----

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])

// EXAMPLE
//
// case class Administrator(department: String) extends Role
// 
// {
//   "admin": { "department": "windmill-tilting" }
// } 
//

val adminRoleAlt = Alt[Role, String](
  "user", 
  JObjT(
    FreeAp.lift[PropSchema[String, ?], String](
      PropSchema("department", JStrT, identity)
    )
  ), 
  Administrator(_),
  { 
    case Administrator(dept) => Some(dept)
    case _ => None
  }
)
~~~

Sum types
----

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])

// EXAMPLE
//
// case object User extends Role
//
// "user": {}
//
~~~

Sum types
----

~~~scala
case class JSumT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B => A, preview: A => Option[B])

// EXAMPLE
//
// case object User extends Role
//
// "user": {}
//

val userRoleAlt = Alt[Role, Unit](
  "user", 
  JObjT(
    FreeAp.pure[PropSchema[Unit, ?], Unit](
      Unit
    )
  ), 
  (_: Unit) => User, 
  { 
    case User => Some(Unit)
    case _ => None
  }
)

val roleSchema: JSchema[Role] = JSumT(userRoleAlt :: adminRoleAlt :: Nil)
~~~

Sum type serialization
----------------------

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    //...
    case JSumT(alts) => 
      val results = alts flatMap {
        case Alt(id, base, _, preview) => 
          preview(value).map(serialize(base, _)).toList map { json =>
            jObject(JsonObject.single(id, json))
          }
      } 

      results.head //yeah, I know
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

Sum type parsing
----------------

~~~scala
def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    //...
    case JSumT(alts) => DecodeJson { (c: HCursor) => 
      val results = for {
        fields <- c.fields.toList
        altResult <- alts flatMap {
          case Alt(id, base, review, _) =>
            fields.exists(_ == id).option(
              c.downField(id).as(decoder(base)).map(review)
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
~~~

What else can we do?
--------------------

* ScalaCheck [Gen](https://github.com/rickynils/scalacheck/blob/master/src/main/scala/org/scalacheck/Gen.scala) instances
* Binary serialization using [scodec](https://github.com/scodec/scodec)
* [User interfaces](https://app-dev.pellucid.com)
* Interpret to whatever Applicative you want, really.

<div class="notes">
Franco Ponticelli, a coworker of mine, wrote a schema interpreter that
he uses to serialize values to database tables... and provides the 'CREATE TABLE'
statements accordingly.
</div>

So... what's the catch?
-----------------------

* Fixed set of primitives is overly limiting
* Unable to provide additional metadata about values/fields without altering the schema algebra

So let's fix these things.

Problem 1: Primitives
---------------------

We started off by defining schema for String, Double, and Boolean. This clearly isn't enough.
For example, I want to be able to define a schema where date values, represented in JSON
as strings, are first-class.

~~~scala

// sealed trait JSchema[A]
// 
// case object JStrT extends JSchema[String]
// case object JNumT extends JSchema[Double]
// case object JBoolT extends JSchema[Boolean]

sealed trait JSchema[P[_], A]

case class JPrimT[P[_], A](pa: P[A]) extends JSchema[P, A]

sealed trait ToJson[S[_]] {
  def serialize(schema: S[A], value: A): Json
}

implicit class JSchemaToJson[P[_]: ToJson] extends ToJson[JSchema[P, ?]] {
  def serialize[A](schema: JSchema[P, A], value: A): Json = ???
}
~~~
