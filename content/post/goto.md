---
title: "Creating a GOTO Monad in Scala"
date: 2017-10-26T20:39:08+01:00
draft: true
---

## The inspiration for this madness
When using Postman to test some APIs recently it occurred to me that
the chain of API calls I'd built up was a lot like a for comprehension 
in Scala, but the rudimentary control flow logic available in Postman
meant I was able to arbitrarily jump to any previous step on failure.

This lead to me wondering whether it is possible to create a Monad
to allow for-comprehensions in Scala to execute upwards as well as downwards?

Take this simple chain of API calls as an example

```scala
/**
 * Hits a bunch of API endpoints, one after another
 * to dig out the nested bit of information we require
 */
def fetchPaymentCard: Future[PaymentCard] = 
  for {
    customer <- getCustomer()                   // Future[Customer]
    account <- getAccount(customer)             // Future[Account]
    subscription <- getSubscription(account)    // Future[Account]
    paymentCard <- getPaymentCard(subscription) // Future[PaymentCard]
  } yield paymentCard

```

The thread of execution here is always going to proceed down the for-comprehension,
executing each step as it goes until either a step fails or it finishes everything.

Imagine though if on failure we could *go back up* so we could retry from an
earlier point in time, without actually jumping out of the for-comprehension:

```scala
/**
 * Hits a bunch of API endpoints, one after another
 * to dig out the nested bit of information we require
 */
def fetchPaymentCard: Future[PaymentCard] = 
  for {
    customer <- getCustomer()                   // ok    
    account <- getAccount(customer)             // ok     <---  
    subscription <- getSubscription(account)    // ok        | GOTO getAccount
    paymentCard <- getPaymentCard(subscription) // fails! >--- 
  } yield paymentCard
```

Now any reasonable Scala developer will be able to give you numerous ways to
retry failed computations using `recoverWith` or a similar function, but I thought
it'd be fun to try and build this retry logic into a new Monad so it "just works"
with minimal acrobatics.

---------------------

## Designing a new Monad - The context

Monads are sometimes referred to as 
"the result of a computation within a context", a phrase which 
makes no sense at all until you look at some concrete examples:

 - in an `Option[Int]` the context is that you might not actually have an `Int` at all.
 - in a `List[Int]` the context is that you might have any number of `Ints`
 - In a `Future[Int]` the context is that the `Int` might not be available yet

Essentially the "context" is the reason you're returning some `M[Int]`,
rather than just a good old fashioned `Int`, which is almost certainly what your 
function caller actually wants.

To enable for-comprehensions to be traversed upwards as well as downwards I figured
that each step needs to return its result within a context representing the 
following possibilities

- The computation has succeeded, maybe has a label and we should go forward
- The computation has failed & wants to retry from a previously seen label.

This can be easily modelled in Scala with like so:

```scala
sealed trait Goto[+L, +V]
case class Forward[L, V](label: Option[L], value: V) extends Goto[L, V]
case class Back[L](to: L) extends Goto[L, Nothing]
```

But this is not a Monad yet. Here we've simply created some plain data types,
we haven't specified how to wire together computations returning these types.

---------------------

## Designing a new Monad - the functions

For a type `M[_]` to form a Monad it needs the following functions

- `pure` which lifts a value `A` into an `M[A]`
- `flatMap` which sequences two computations to produce a final result
