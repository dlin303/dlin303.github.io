---
layout: post
title: "Fun times with Phantom Types (Scala Edition) - Part 2"
date: 2016-11-06
categories: Scala Type-Programming
---

The 'Double Edit Sword' aka 'What in the world is =:='
=====================================================

So this type `A =:= A` (which can also be written `=:=[A, A]`) is defined in the scala api here: [https://github.com/scala/scala/blob/v2.11.8/src/library/scala/Predef.scala#L403][scala-api], which we will be referencing, put will paste here for convenience:

{% highlight scala %}
  @implicitNotFound(msg = "Cannot prove that ${From} =:= ${To}.")
  sealed abstract class =:=[From, To] extends (From => To) with Serializable

  ...

  object =:= {
     implicit def tpEquals[A]: A =:= A = singleton_=:=.asInstanceOf[A =:= A]
  }
{% endhighlight %}

The first important thing to note, is that the `=:=` type is defined as `=:=[From, To]`, making it invariant.

What this means is that the implicit method `implicit def tpEquals`, returns a type of `A =:= A` (i.e. `=:=[A, A]`) where the two `A`'s MUST be exactly the same type.
If `=:=` was defined as covariant, (i.e. `=:=[+From, -To]`), then `tpEquals` could return `A =:= A` where the actual types of the two `A`'s could both be subclasses of `A`.
But `=:=` is defined as invariant so the compiler doesn't allow that.

**So how is this super useful tidbit of info used?**

In the method signature of `cancelSub` (see post: {% post_url 2016-11-06-fun-times-phantom-types %}), we've asked the compiler for evidence that `S =:= Active`. 

{% highlight scala %}
def cancelSub[S <: Status](sub: Sub[S])(implicit evidence: S =:= Active): Sub[Cancelled]
{% endhighlight %}

Therefore, wherever `cancelSub` is invoked, the compiler searches it's implicit scope
for an implicit value of type `=:=[Active, Active]`. 

Since `S` is actually `Active`, we are able to find this value by invoking the implicit method `tpEquals` which has a type signature of `A =:= A`.

Again, since the type =:= is defined as invariant, the two A's in tpEquals method signature must be of the EXACT same type.

In this case, `S` is actually `Active` so this condition is satisfied, and `=:=[Active, Active]` is found. (i.e. `tpEquals[Active] => =:=[Active, Active]`)

However, for `cancel(Sub[Cancelled](123))`, the compiler is unable to find an implicit value `=:=[Active, Active]`. Again, this is because `=:=` is invariant. The `implicit def tpEquals` is not invoked
since it's return type of `A =:= A` cannot be satisfied since `Cancelled` and `Active` are not the exact same type (despite both being subclasses of `Status`).

In other words, although `=:=[Cancelled, Active]` is a valid type, it does not satisfy `tpEquals` requirement of `A =:= A`.

[scala-api]: https://github.com/scala/scala/blob/v2.11.8/src/library/scala/Predef.scala#L403
