% Describing Data
% Kris Nuttycombe (@nuttycom)
% September, 2017

Goals
-----

* Explore the design of an FP library from start to finish
* Introduce **free applicative functors** in the context of a "real-world" problem
* Discuss how **fix-point types** allow us to annotate recursive data structures
* See how these things come together in the [**xenomorph**](https://github.com/nuttycom/xenomorph) library

Overview
--------

* This talk is about the design of a library using pure FP style.
* Dissatisfaction-Driven Design
* Problem: Serialization
    * Having to maintain both serializers and deserializers is silly
    * Problems exist with macros/generic programming approaches 
    * Legacy wire formats & evolving protocols can present challenges.
* Solution:
    * Build a description of the data structure that can be interpreted
      to derive serializers, deserializers, and more.
    * When faced with a problem, describe that problem completely. 
      The solution usually resides within that description.
* Conclusion:
    * Abstractive capability in programming languages & its uses

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

Prerequisites
-------------

* Terminology
    * Sum Types / GADTs
    * Product Types
    * Applicative Functors
    * Higher-kinded types
    * Coproducts
* Tools
    * Type-level lambdas / kind-projector
    * Natural transformations
    * Lenses & Prisms

Example
-------

A simple sums-of-products data type.

~~~scala
case class Person(
  name: String, 
  birthDate: DateTime,
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
  birthDate: DateTime,
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
  birthDate: DateTime,
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
  birthDate: Instant,
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
  birthDate: Instant,
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

Example JSON Representation
---------------------------

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

Example Schema
--------------
 
~~~scala
val personSchema = Schema.rec[Prim, Person](
  ^^[TProp[Person, ?], String, Instant, Vector[Role], Person](
    required("name", Prim.str, Person.name.asGetter),
    required("birthDate", Prim.long, Getter.id[Long])).dimap(
      (_: Person).birthDate.getMillis,
      new Instant(_: Long)
    ),
    required("roles", Prim.arr(roleSchema), Person.roles.asGetter)
  )(Person.apply _)
)
~~~

Example Schema
--------------
 
~~~scala
val personSchema = Schema.rec[Prim, Person](
  ^^[TProp[Person, ?], String, Instant, Vector[Role], Person](
    required("name", Prim.str, Person.name.asGetter),
    required("birthDate", Prim.long, Getter.id[Long])).dimap(
      (_: Person).birthDate.getMillis,
      new Instant(_: Long)
    ),
    required("roles", Prim.arr(roleSchema), Person.roles.asGetter)
  )(Person.apply _)
)
~~~

~~~scala
val roleSchema = Schema.oneOf(
  alt[Unit, Prim, Role, Unit](
    "user", 
    Schema.empty,
    Role.user composeIso GenIso.unit[User.type]
  ) ::
  alt[Unit, Prim, Role, Administrator](
    "admin", 
    rec[Prim, Administrator](
      required("department", Prim.str, Administrator.department.asGetter) map {
        Administrator.apply _
      }
    ),
    Role.admin
  ) :: Nil
)
~~~

Primitives
----------

Use a GADT to describe the kinds of elements that can exist.

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
~~~

Primitive serialization
-----------------------

Use a GADT to describe the kinds of elements that can exist.

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]
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
    case JNumT  => LongDecodeJson
  }
}
~~~

Sequences
---------

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

Sequences
---------

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

~~~scala
val boolsSchema: JSchema[Vector[Boolean]] = JArrayT(JBoolT, 0, None)
~~~

Sequence serialization
-----------------------

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

~~~scala
val boolsSchema: JSchema[Vector[Boolean]] = JArrayT(JBoolT, 0, None)
~~~

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    //...
    case JVecT(elemSchema) => jArray(value.map(serialize(elemSchema, _)).toList)
  }
}
~~~

Sequence parsing
-----------------

Sequences are simple to describe because the only thing you need to represent
is the type of the element. 

~~~scala
case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

~~~scala
val boolsSchema: JSchema[Vector[Boolean]] = JArrayT(JBoolT, 0, None)
~~~

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    //...
    case JVecT(elemSchema) => jArray(value.map(serialize(elemSchema, _)).toList)
  }
}
~~~

~~~scala
def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    //...
    case JVecT(elemSchema) => VectorDecodeJson(decoder(elemSchema))
  }
}
~~~

Records
-------

~~~scala
case class Person(
  name: String, 
  birthDate: Instant 
)
~~~

Records
-------

~~~scala
case class Person(
  name: String, 
  birthDate: Instant
)
~~~

~~~scala
def liftA2[A, B, C, F[_]: Applicative](fa: F[A], fb: F[B])(f: (A, B) => C): F[C]

val personSchema: JSchema[Person] = liftA2(JStrT, JNumT) { Person.apply _ }
~~~

This looks like exactly the sort of thing that we need, but we immediately run
into a problem if we try to write Applicative[JSchema]. We can't even make JSchema
a functor!

Records
-------

~~~scala
sealed trait JSchema[A]

case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Long]
case object JBoolT extends JSchema[Boolean]

case class JVecT[A](elemType: JSchema[A]) extends JSchema[Vector[A]]
~~~

Records, Take 2
---------------

We need an applicative functor, but maybe not for the whole schema type.
What about just for the product types?

~~~scala
case class JObjT[A](props: Props[A]) extends JSchema[A]
~~~

Records, Take 2
---------------

We need an applicative functor, but maybe not for the whole schema type.
What about just for the product types?

~~~scala
case class JObjT[A](props: Props[A]) extends JSchema[A]
~~~

~~~scala
def liftA2[A, B, C](fa: Props[A], fb: Props[B])(f: (A, B) => C): Props[C]

// or, isomorphically,

def ap[A, B](fa: Props[A], f: Props[A => B]): Props[B]
~~~

Records, Take 2
---------------

To define our record builder first define a class that captures
the name and the schema for a single property. 

~~~scala
case class JObjT[O](props: Props[O]) extends JSchema[O]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 2
---------------

To define our record builder first define a class that captures
the name and the schema for a single property. 

~~~scala
case class JObjT[O](props: Props[O, O]) extends JSchema[O]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Now, wrap **that** up in a data structure for which we can define the applicative
operations.

~~~scala
sealed trait Props[O, A] 

case class PureProps[O, A](a: A) extends Props[O, A]

case class ApProps[O, A, B](prop: PropSchema[O, B], rest: Props[O, B => A]) extends Props[O, A]
~~~

Records, Take 2
---------------

Is it applicative? We can find out by implementing `ap`.

~~~scala
def applicative[O] = new Applicative[Props[O, ?]] {
  def point[A](a: => A): Props[O, A] = PureProps(a)

  override def map[A,B](fa: Props[O, A])(f: A => B): Props[O, B] = {
    fa match {
      case PureProps(a) => PureProps(f(a))
      case ApProps(prop, rest) => ApProps(prop, map(rest)(f compose _))
    }
  }

  def ap[A,B](fa: => Props[O, A])(ff: => Props[O, A => B]): Props[O, B] = {
    ff match {
      case PureProps(f) => map(fa)(f)
      case aprb: ApProps[O, (A => B), i] => 
        ApProps(
          aprb.prop, 
          ap(fa) { 
            map[i => (A => B), A => (i => B)](aprb.rest) { 
              (g: i => (A => B)) => { (a: A) => { (i: i) => g(i)(a) } } // this is just flip
            }
          }
        )
    }
  }
}
~~~

<div class="notes">
So, is this thing applicative? Well, the easiest way to find out is to go ahead and
see if you can just implement the Applicative typeclass in a way that makes sense.
</div>

Records, Take 2
---------------

~~~scala
case class JObjT[A](props: Props[PropSchema, A, A]) extends JSchema[A]

sealed trait Props[F[_, _], O, A] 

case class ApProps[F[_, _], O, A, B](
  prop: F[O, B], 
  rest: Props[F, O, B => A]
) extends Props[F, O, A]

case class PureProps[F[_, _], O, A](a: A) extends Props[F, O, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 2
---------------

~~~scala
case class JObjT[A](props: Props[PropSchema[A, ?], A]) extends JSchema[A]

sealed trait Props[F[_], A] 

case class ApProps[F[_], A, B](
  prop: F[B], 
  rest: Props[F, B => A]
) extends Props[F, A]

case class PureProps[F[_], A](a: A) extends Props[F, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 3
---------------

~~~scala
case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]

sealed trait FreeAp[F[_], A] 

case class Ap[F[_], A, B](
  head: F[B], 
  tail: FreeAp[F, B => A]
) extends FreeAp[F, A]

case class Pure[F[_], A](a: A) extends FreeAp[F, A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~

Records, Take 3
---------------

~~~scala
import scalaz.FreeAp

case class JObjT[A](props: FreeAp[PropSchema[A, ?], A]) extends JSchema[A]

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
  "admin", 
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

We started off by defining schema for String, Long, and Boolean. This clearly
isn't enough.  For example, I want to be able to define a schema where date
values, represented in JSON as strings, are first-class.

~~~scala
// sealed trait JSchema[A]
// 
// case object JStrT extends JSchema[String]
// case object JNumT extends JSchema[Long]
// case object JBoolT extends JSchema[Boolean]

sealed trait GSchema[P[_], A]

case class JPrimT[P[_], A](prim: P[A]) extends GSchema[P, A]

case class JVecT[P[_], A](elemType: GSchema[P, A]) extends GSchema[P, Vector[A]]

case class JSumT[P[_], A](alternatives: List[Alt[P, A, B] forSome { type B }]) extends GSchema[P, A]
case class Alt[P[_], A, B](id: String, base: GSchema[P, B], review: B => A, preview: A => Option[B])

case class JObjT[P[_], A](props: FreeAp[PropSchema[P, A, ?], A]) extends GSchema[P, A]
case class PropSchema[P[_], O, A](fieldName: String, valueSchema: GSchema[P, A], accessor: O => A)
~~~

Problem 1: Primitives
---------------------

Then, we define a separate GADT just defining the primitive types we're
interested in.

~~~scala
sealed trait JsonPrim[A]
case object JStrT extends JsonPrim[String]
case object JNumT extends JsonPrim[Long]
case object JBoolT extends JsonPrim[Boolean]
~~~

Problem 1: Primitives
---------------------

Then, we define a separate GADT just defining the primitive types we're
interested in.

~~~scala
sealed trait JsonPrim[A]
case object JStrT extends JsonPrim[String]
case object JNumT extends JsonPrim[Long]
case object JBoolT extends JsonPrim[Boolean]
~~~

~~~scala
type JSchema[A] = GSchema[JsonPrim, A]
~~~

Problem 1: Primitives
---------------------

In order to define serialization in the presence of this new layer
of abstraction, we're going to need a bit more machinery:

~~~scala
sealed trait ToJson[S[_]] {
  def serialize[A](schema: S[A], value: A): Json
}

implicit def jSchemaToJson[P[_]: ToJson] = new ToJson[JSchema[P, ?]] {
  def serialize[A](schema: JSchema[P, A], value: A): Json = {
    schema match {
      case JPrimT(p) => implicitly[ToJson[P]].serialize(p, value)
      // handling the other constructors stays the same
    }
  }
}

implicit val JsonPrimToJson = new ToJson[JsonPrim] {
  def serialize[A](p: JsonPrim[A], value: A): Json = {
    schema match {
      case JStrT  => jString(value)
      case JNumT  => jNumber(value)
      case JBoolT => jBool(value)
    }
  }
}
~~~

Problem 1: Primitives
---------------------

Problem 1: Primitives
---------------------

Having done this, we get an added bonus: We can use coproducts
to compose sets of primitives!

~~~scala
  implicit def primCoToJson[P[_]: ToJson, Q[_]: ToJson] = new ToJson[Coproduct[P, Q, ?]] {
    def serialize[A](p: Coproduct[P, Q, A], value: A): Json = {
      p.run.fold(
        implicitly[ToJson[P]].serialize(_, value),
        implicitly[ToJson[Q]].serialize(_, value)
      )
    }
  }

  implicit def primCoFromJson[P[_]: FromJson, Q[_]: FromJson] = new FromJson[Coproduct[P, Q, ?]] {
    def decoder[A](p: Coproduct[P, Q, A]): DecodeJson[A] = {
      p.run.fold(
        implicitly[FromJson[P]].decoder(_),
        implicitly[FromJson[Q]].decoder(_)
      )
    }
  }
~~~

Problem 2: Annotations
----------------------

* Nodes of the JSchema tree don't currently contain enough
  information to generate interesting json-schema.
    * titles
    * min/max length of arrays
    * property order
    * formatting metadata
    * etc.
* We might want different kinds of metadata for different
  applications.

Problem 2: Annotations
----------------------

~~~scala
sealed trait JSchema[P[_], I]

case class JPrimT[P[_], I](prim: P[I]) extends JSchema[P, I]

case class JVecT[P[_], I](elemType: JSchema[P, I]) extends JSchema[P, Vector[I]]

case class JSumT[P[_], I](alternatives: List[Alt[P, I, J] forSome { type J }]) extends JSchema[P, I]
case class Alt[P[_], I, J](id: String, base: JSchema[P, J], review: J => I, preview: I => Option[J])

case class JObjT[P[_], I](props: FreeAp[PropSchema[P, I, ?], I]) extends JSchema[P, I]
case class PropSchema[P[_], O, I](fieldName: String, valueSchema: JSchema[P, I], accessor: O => I)
~~~

Problem 2: Annotations
----------------------

~~~scala
sealed trait JSchema[A, P[_], I]

case class JPrimT[A, P[_], I](ann: A, prim: P[I]) extends JSchema[A, P, I]

case class JVecT[A, P[_], I](ann: A, elemType: JSchema[A, P, I]) extends JSchema[A, P, Vector[I]]

case class JSumT[A, P[_], I](ann: A, alternatives: List[Alt[A, P, I, J] forSome { type J }]) extends JSchema[A, P, I]
case class Alt[A, P[_], I, J](id: String, base: JSchema[A, P, J], review: J => I, preview: I => Option[J])

case class JObjT[A, P[_], I](ann: A, props: FreeAp[PropSchema[A, P, I, ?], I]) extends JSchema[A, P, I]
case class PropSchema[A, P[_], O, I](fieldName: String, valueSchema: JSchema[A, P, I], accessor: O => I)
~~~

This is a bit messy and involves a bunch of redundancy. It will work; it just doesn't seem ideal.

Directly recusive data
----------------------

~~~scala
class Prof(
  name: String,
  students: List[Prof]
)
~~~


No longer directly recusive data
--------------------------------

~~~scala
class ProfF[S](
  name: String,
  students: List[S]
)
~~~

Reintroducing recursion with Fix
--------------------------------

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

Annotating a tree with Cofree
-----------------------------

~~~scala
class ProfF[S](
  name: String,
  students: List[S]
)
~~~

~~~scala
case class Cofree[F[_], A](f: F[Cofree[F, A]], a: A)

type IdProf = Cofree[Prof, Int]
~~~

Hat tip to Rob Norris, go watch his talk [here](https://www.youtube.com/watch?v=7xSfLPD6tiQ)

Annotating a tree with Cofree
-----------------------------

~~~scala
sealed trait JSchema[P[_], I]

case class JVecT[P[_], I](elemType: JSchema[P, I]) extends JSchema[P, Vector[I]]
~~~

Annotating a tree with Cofree
-----------------------------

~~~scala
sealed trait JSchema[S, P[_], I]

case class JVecT[S, P[_], I](elemType: S) extends JSchema[S, P, Vector[I]]
~~~

Does this work?

Annotating a tree with ~~Cofree~~ HCofree
-----------------------------------------

~~~scala
sealed trait JSchema[F[_], P[_], I]

case class JVecT[F[_], P[_], I](elemType: F[I]) extends JSchema[F, P, Vector[I]]
~~~

Annotating a tree with ~~Cofree~~ HCofree
-----------------------------------------

~~~scala
sealed trait JSchema[F[_], P[_], I]

case class JVecT[F[_], P[_], I](elemType: F[I]) extends JSchema[F, P, Vector[I]]
~~~

~~~scala
case class HCofree[F[_[_], _], A, I](f: F[HCofree[F, A, ?], I], a: A)
~~~

Annotating a tree with ~~Cofree~~ HCofree
-----------------------------------------

~~~scala
sealed trait JSchema[F[_], P[_], I]

case class JVecT[F[_], P[_], I](elemType: F[I]) extends JSchema[F, P, Vector[I]]
~~~

~~~scala
case class HCofree[F[_[_], _], A, I](f: F[HCofree[F, A, ?], I], a: A)
~~~

~~~scala
type Schema[P, A] = HCofree[Schema[?[_], P, ?], A]
~~~

Drawbacks
---------

The Scala compiler hates me.

~~~
[error] /Users/nuttycom/personal/scala_world-2017/sample_code/xenomorph/src/main/scala/xenomorph/Schema.scala:263: type mismatch;
[error]  found   : [γ$13$]xenomorph.PropSchema[O,[γ$40$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$13$] ~> [γ$14$]xenomorph.PropSchema[N,[γ$40$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$14$]
[error]     (which expands to)  scalaz.NaturalTransformation[[γ$13$]xenomorph.PropSchema[O,[γ$40$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$13$],[γ$14$]xenomorph.PropSchema[N,[γ$40$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$40$],γ$14$]]
[error]  required: [γ$3$]xenomorph.PropSchema[O,[γ$2$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$2$],γ$3$] ~> G
[error]     (which expands to)  scalaz.NaturalTransformation[[γ$3$]xenomorph.PropSchema[O,[γ$2$]xenomorph.HCofree[[β$0$[_$1], γ$1$]xenomorph.SchemaF[P,β$0$,γ$1$],A,γ$2$],γ$3$],G]
[error]       PropSchema.contraNT[O, N, Schema[A, P, ?]](f)
~~~

That's the result of leaving off a type ascription, there's actually nothing incorrect about the code.
