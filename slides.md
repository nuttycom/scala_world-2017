
============================

Kris Nuttycombe

(@nuttycom)

September, 2017

<aside class="notes">

</aside>

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


~~~{scala}



case class 

~~~

Describe a data structure
-------------------------

* Primitives
* Sequences
* Sums
* Products

Primitives
----------

~~~{scala}

sealed trait JsonT[A]

case object JBoolT extends JsonT[Boolean]
case object JStrT extends JsonT[String]
case object JNumT extends JsonT[Double]

~~~

Sequences
---------

~~~{scala}

case class JArrayT[A](elemType: JsonT[A]) extends JsonT[Vector[A]]

~~~

Sums
----

~~~{scala}

case class JOneOfT[A](alternatives: List[Alt[A, B] forSome { type B }]) extends JsonT[A]

case class Alt[A, B](id: String, base: JsonT[B], review: B -> A, preview: A -> Option[B])

~~~


* Freeness:
  * Free monads describe the structure of a computation
  * Free applicative functors describe how parts of a data structure relate
    to one another




------

<img src="./img/fp_with_values.jpg" width="800"/>

<aside class="notes">

</aside>


=====

<aside class="notes">

</aside>


========

<aside class="notes">

</aside>

---------

<div class="fragment">

> 
> 

</div>
