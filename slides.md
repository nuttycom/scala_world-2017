% Describing Data
% Kris Nuttycombe (@nuttycom)
% September, 2017

Goals
-----

* Introduce **free applicative functors** in the context of a "real-world" problem
* Discuss how **fix-point types** allow us to annotate recursive data structures
* See how these things come together in the **schematic** library
* Explore the design of an FP library from start to finish
* Make hard things easy

Outline
-------

* This talk is about design of a library in functional programming
* Composition of talks, Rob Norris and John De Goes
  * What I'm presenting here are not really new ideas, just taking
    some existing ideas and composing them in a slightly new way
    - dissatisfaction driving design
* Problem: Serialization
  * Having to maintain both serializers and deserializers
  * Problems with generic programming approaches - serialized
    form becomes coupled to the type being represented, making
    it harder to change things.
  * Difficulties in dealing with legacy wire formats
* Solution:
  * Build a description of the data structure that can be interpreted
    to derive serializers, deserializers, and more.

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

Components of a data structure
------------------------------

* Primitives
* Sequences
* Sums
* Records

Primitives
----------

Descriptions of primitive types

~~~scala

sealed trait JSchema[A]

case object JBoolT extends JSchema[Boolean]
case object JStrT extends JSchema[String]
case object JNumT extends JSchema[Double]

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

Primitive parsing
-----------------

Create a parser by generating a natural transformation between
a schema and an argonaut DecodeJson instance.

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

Sums
----

~~~scala
case class JOneOfT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JSchema[A]

case class Alt[A, B](id: String, base: JSchema[B], review: B -> A, preview: A -> Option[B])
~~~

Sum type serialization
----------------------

~~~scala
def serialize[A](schema: JSchema[A], value: A): Json = {
  schema match {
    //...
    case JOneOfT(alts) => 
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

Sum type parsing
----------------

~~~scala
def decoder[A](schema: JSchema[A]): DecodeJson[A] = {
  schema match {
    //...
    case JOneOfT(alts) => DecodeJson { (c: HCursor) => 
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

case class JObjT[A](recordBuilder: FreeAp[({ type l[a] = PropSchema[A, a] })#l, A]) extends JSchema[A]

case class PropSchema[O, A](fieldName: String, valueSchema: JSchema[A], accessor: O => A)
~~~
What else can we do?
--------------------

* Gen instances
* User interfaces (Pellucid demo)
* ???
