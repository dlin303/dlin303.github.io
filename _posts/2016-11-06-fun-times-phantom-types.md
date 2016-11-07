---
layout: post
title: "Fun times with Phantom Types (Scala edition)"
date: 2016-11-06
categories: Scala Type-Programming
---

Fun times with Phantom Types (Scala edition)
============================================
**What is a phantom type?**

A type that is never actually instantiated.


**Cool! What does that look like?**

Phantom types often appear as type parameters to a class, but which are not used actually used by the class itself.


**Awesome! That sounds like it could be really useless!**

Not really. Phantom types allow other code to reason about instances of the class with the added security of compile time type safety.


**Cool! What does that look like?** 

Ok fine let's code.


Let's take a toy example of an object oriented representation of some subscription Sub with only two fields: an ID and a status. 
In our example, a subscription can only contain one of two statuses, Active, or Cancelled.

{% highlight scala %}
sealed trait Status
case object Active extends Status
case object Cancelled extends Status

case class Sub(id: Long, status: Status)
{% endhighlight %}

At some point, we'd probably like to write some business method for cancelling an active subscription. 

A simple implementation would be: 

{% highlight scala %}
def cancelSub(sub: Sub): Sub = {
    //do a bunch of business-y things
    sub.copy(status = Cancelled)
} 
{% endhighlight %}


This method is dumb for a variety of reasons, but the one reason we'll focus on today is the fact that you shouldn't be allowed to cancel a sub that is already cancelled.
Let's make our method a little smarter by checking for a Sub's status before cancelling it.

{% highlight scala %}
def cancelSub(sub: Sub): Option[Sub] = {
    sub.status match {
        case Cancelled => 
            //can't cancel already cancelled sub!
            None 
        case Active => 
            //do a bunch of business-y things
            sub.copy(status = Cancelled)
    }
} 
{% endhighlight %}

The above code snippet is fine and satisfies our main requirement: not allowing an already `Cancelled` sub to be cancelled. 
However, once we start expanding the number of business methods that are dependent on the state of `Sub`, it quickly becomes tedious to keep checking and rechecking the status of `Sub`. 

Instead, we can let the compiler do the heavy lifting for us by introducing phantom types on `Sub`.

Let's redefine our statuses as traits instead of objects and also redefine Sub to take in a type parameter for it's status:

{% highlight scala %}
sealed trait Status
sealed trait Active extends Status
sealed trait Cancelled extends Status

case class Sub[S <: Status](id: Long)
{% endhighlight %}

Now we can rewrite our cancelSub method like so:

{% highlight scala %}
def cancelSub[S <: Status](sub: Sub[S])(implicit evidence: S =:= Active): Sub[Cancelled] = {
    //do business-y stuff
    Sub[Cancelled](sub.id) //probably a better way to do this?
}
{% endhighlight %}

Essentially what we've done is defined `cancelSub` to take in a parameter of type `Sub[S]`, where `S` is a subtype of `Status`. Then, using the weird `=:=[From, To]` type, we ask the 
compiler for evidence that `S` is of type `Active`. Now, if this method is passed a `Sub[Cancelled]`, it will fail to compile.

{% highlight scala %}
cancelSub(Sub[Cancelled](123))
error: Cannot prove that Cancelled =:= Active.
       cancelSub(Sub[Cancelled](123))
{% endhighlight %}

However... 
`cancelSub(Sub[Active](123))` will compile and run just fine. 

Thus, by introducing phantom types to Sub, we've lifted the enforcement of valid operations from manual checking performed at runtime, to statically checked at compile time. 

Side Note: 
You might expect `cancelSub(Sub(123))` to fail compilation since you would expect `Sub(123)`  to return a `Sub[Nothing]`. This is true if you were to declare it in a vacuum, e.g.
`val nothing = Sub[Nothing]`

However, by some weird type inferencing thing which I haven't bothered to examine (yet), `cancelSub(Sub(123))` actually gets instantiated as a `Sub[Active]`. 
Go figure.


(check out {% post_url 2016-11-06-fun-times-phantom-types-part-2 %} for a half-baked explanation of what the hell `=:=` is)
