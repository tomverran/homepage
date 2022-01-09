---
title: How I write Scala in 2022
date: 2022-01-09
---

Historically Scala has been infamous for being so expressive that everyone ends up writing
their own dialect, the result being that interoperability between libraries can be an issue
and onboarding new developers can be tricky. Things have mercifully calmed down a lot recently
but I still thought it'd be interesting to share how I like to write software with Scala in 2022.

<!--more-->

### Disclaimer

This is going to be a very opinionated article so I want to make it clear that I'm not 
suggesting that _everyone_ should do these things, or even that all of these ideas are good!
I am looking forward to coming back in a few years and seeing how much of this article I now disagree with.

The code examples provided here are essentially psuedocode because I didn't try compiling any of them.
It also goes without saying that very few of these conventions are truly _mine_ per se, rather they're things 
that over the years I have learned from sensible people.

### Libraries

I'm a big fan of the [Typelevel stack](https://typelevel.org) so have a fairly prescriptive set
of libaries I reach for when building any applications:

- [Cats](https://github.com/typelevel/cats) and [Cats Effect](https://typelevel.org/cats-effect/) for functional programming primitives.
- [Doobie](https://github.com/tpolecat/doobie) for connecting to Postgres, soon to be replaced with [Skunk](https://github.com/tpolecat/skunk).
- [Log4cats](https://github.com/typelevel/log4cats) backed by logback for logging.
- [MUnit](https://github.com/scalameta/munit) and [Scalacheck](https://github.com/typelevel/scalacheck) for testing.
- [FS2](https://github.com/typelevel/fs2) for concurrency & streaming.
- [FS2 Kafka](https://github.com/fd4s/fs2-kafka) and [Vulcan](https://github.com/fd4s/vulcan) for Kafka and Avro.
- [Natchez](https://github.com/tpolecat/natchez) for tracing.
- [http4s](https://github.com/http4s/http4s) for http.

### Build tools

I keep it simple here and use SBT. I've had some experiences with Bazel but I like to expend
as little effort thinking about build tools as possible and I find it is easiest to follow the crowd.
I try to keep the number of SBT plugins low so that they're unlikely to prevent upgrades and to keep
the scope of what SBT does fairly limited.

### Module structure

I'm not very imaginative with application architecture. I tend to structure applications by
stitching together the following types of module, where a "module" is a Scala `object` or `trait`.

- `Client`: HTTP clients calling external APIs with very little extra logic.
- `Storage`: Modules that define database queries to store & retrieve things.
- `Service`: Modules that apply application logic on top of `Storage` or `Client`
- `Routes`: HTTP routes that call out to `Services` and format their responses.
- `Consumer`: Kafka consumers that also call out to other modules to process messages.
- `Publisher`: Background processes that relay information from the database to third parties.

I try to keep the core logic of the application as separate from any IO as possible so
will often also have modules consisting of objects that contain 
[pure functions](https://docs.scala-lang.org/overviews/scala-book/pure-functions.html). An example of this
might be functions to determine whether a customer is eligible to sign up to a product based
on the state of their account. A `Service` would collect together the state from `Storage` objects
and pass it along to the pure functions that make the decisions.

This very formulaic process of following a strict recipe to create an application 
is something historically I didn't like doing because it felt like it took the creativity out of the job 
but I like that having strong conventions takes (most of) the guesswork out of where 
to put things or what to call them. Breaking these conventions when it makes sense 
is, however, encouraged.

### Directory structure

I organise the packages in each application by data (like customers, accounts)
rather than layer (services, storage) so it is easy to see at a glance 
exactly what kind of things the application does. Here's an example for a hypothetical 
event booking application that allows events to be  created & signed up for at venues which 
are consumed from Kafka:

```
src/main/scala/uk/tomverran/
 |_ events/
    |_ data.scala
    |_ EventRoutes.scala
    |_ EventService.scala
    |_ EventStorage.scala
 |_ customers/
    |_ data.scala
    |_ CustomerRoutes.scala
    |_ CustomerService.scala
    |_ CustomerStorage.scala
 |_ venues/
    |_ data.scala
    |_ VenueConsumer.scala
    |_ VenueStorage.scala
```

A nice property of this structure is that as the application grows it will tend to do so horizontally,
accumulating more directories that correspond to more features, rather than vertically through files
becoming longer or directories accumulating more files.

### Code structure

Each `Storage` or `Service` module consists of a trait and an anonymous implementation in the same file.
`Storage` modules are implemented in terms of `ConnectionIO` which means that multiple operations
across multiple storage modules can be combined to occur to within one database transaction.

```scala
trait CustomerStorage[F[_]] {
  def store(customer: Customer): F[Unit]
  def find(customerId: Customerid): F[Option[Customer]]
}

object CustomerStorage[F[_]] {
  val instance: CustomerStorage[ConnectionIO] =
    new CustomerStorage[ConnectionIO] {
      def store(customer: Customer): ConnectionIO[Unit] = ???
      def find(customerId: CustomerId): ConnectionIO[Option[Customer]] = ???
    }  
}
```

Kafka consumers typically are structured as an fs2 `Stream` enclosed in an object, so that
dependencies can be passed like so:

```scala
object VenueConsumer {

  def apply[F[_]](
    kafka: KafkaConsumer[F, Venue],
    venues: VenueStorage[F]
  ): Stream[F, Unit] =
    kafka
      .stream
      .evalTap(message => venues.store(message.content))
      .evalMap(_.offset.commit)
}
```

Common data structures live in `data.scala`. HTTP requests and responses will _usually_ have
their own data structures but I'm fairly relaxed about using a common `case class` across the layers within a package
if all the fields happen to be identical as long as it is easy to split them when they diverge.

### Unit tests

Absolutely everything other than `Main.scala` should have meaningful unit tests.
I don't often write integration tests and when I do I still use a unit test framework.
Nothing makes my heart sink more than updating a lot of unit tests only to discover
that the same functionality is also tested five times more slowly in a bunch of neglected e2e tests that
fail 50% of the time and consume all my RAM.

An example test directory for the above booking application would look like this:

```
src/test/scala/uk/tomverran/
 |_ customers/
    |_ CustomerRoutesTest.scala
    |_ CustomerServiceTest.scala
    |_ TestCustomerService.scala
    |_ CustomerStorageTest.scala
    |_ TestCustomerStorage.scala
```

I do my very best to keep each test isolated by only ever using mock, in-memory implementations of any 
dependencies in tests. These mock implementations are always called `TestXXX.scala` and live alongside 
the tests for the real implementations.

Most of the core application logic should be tested without needing to use any mocks due to it consisting of
objects that contain pure functions. The mocks are used when testing how the various modules are wired 
together. These tests usually cover things like error handling, retry policies and similar.

In the above example `CustomerRoutesTest` will use the mock implementation of `CustomerService` called 
`TestCustomerService` while `CustomerServiceTest` will test the real implementation of `CustomerService` 
using the `TestCustomerStorage` mock. Mock implementations of modules should have as little logic as possible
but it often isn't possible to eliminate all logic and I tend not to worry too much about _some_ logic in mocks.

All this mocking of course has a cost but it keeps the tests fast & focused.
The goal is to keep the friction of writing unit tests as low as possible by always having a mock implementation
of any given module available with a fairly consistent feature set.
 
Mocks of stateful modules are typically backed by a cats-effect `Ref` and will also 
often contain extra helper functions, for example to record which functions were called:

```scala
trait TestCustomerStorage[F[_]] extends CustomerStorage[F] {
  def calls: F[List[TestCustomerStorage.Call]]
}

object TestCustomerStorage {

  sealed trait Call

  object Call {
    case class Store(customer: Customer) extends Call
    case class Find(customerId: CustomerId) extends Call
  }

  def apply[F[_]: Sync]: F[TestCustomerStorage[F]] =
    (
      Ref.of[F, List[Call]](List.empty),
      Ref.of[F, List[Customer]](List.empty)
    ).mapN { (calls, customers) =>
      new TestCustomerStorage[F] {
        def find(customerId: CustomerId): F[Customer] =
          calls.set(_ :+ Call.Find(customerId)) >>
          customers.get.map(_.find(_.id == customerId))
        def store(customer: Customer): F[Unit] =
          calls.set(_ :+ Call.Store(customer)) >>
          customers.modify(_ :+ customer)
        def calls: F[List[Call]] =
          calls.get
      }
    }
}
```

### Property based tests

While I like doing property based testing I don't think I'm very good at it.
I often find that rather than writing true property based tests I use 
[generators](https://github.com/typelevel/scalacheck/blob/main/doc/UserGuide.md#generators) to let 
me randomise fields I don't care about in unit tests and then set one specific field to a 
known value, as in a normal example-based unit test.

Scalacheck provides both `Arbitrary` and `Gen` data structures for generators, I tend
to exclusively use `Gen` just to keep life simple.

When writing generators for data structures I strictly follow the rule that they must
have the same name as the data structure they're generating instances of and they must go
in the same package as the data structure, nested within a `Generators` object due to Scala 2
not allowing package level `val`s.

Sometimes it is useful to have generators that aren't completely random and these should
always be named _very_ explicitly so that it is clear what each test using them is assuming
about the input data. Nothing is worse than changing a generator only to discover that 
50 tests were assuming a particular field would contain a particular set of values.

For example:

`src/main/scala/uk/tomverran/customers/data.scala`

```scala
package uk.tomverran.customers
case class Customer(name: String, age: Int)
```

`src/test/scala/uk/tomverran/customers/Generators.scala`

```scala
object Generators {

  val customer: Gen[Customer] =
    for {
      name <- Gen.alphaNumStr
      age  <- Gen.chooseNum(0, 100)
    } yield Customer(name, age)

  val over60Customer: Gen[Customer] =
    for {
      name <- Gen.alphaNumStr
      age  <- Gen.chooseNum(60, 100)
    } yield Customer(name, age)
}
```

I don't think this directory structure is a particularly _good_ convention but I do think that it is important
to have some kind of convention for where to define generators otherwise it is easy to end up with every unit test
re-implementing a bunch of generators and that hurts a lot when adding or removing fields.

### Examples

[citysocials-api](https://github.com/tomverran/citysocials-api)
was a side project I briefly took on in 2020 to make a Meetup clone.
I gave up on it very quickly so it doesn't have much implemented but it was the first project where I
applied some of these principles so might be a useful source of further code examples.