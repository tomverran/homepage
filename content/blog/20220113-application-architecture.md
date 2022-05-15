---
title: How I architect software in 2022
date: 2022-01-13
draft: true
---


Architecture can feel a very grandiose term when describing how to assemble software,
I struggle to describe anything I do as architecture for the same reasons that I
struggle to describe myself as a chef when adjusting the dial on my toaster.

Nevertheless I find that over the years I've ended up with a lot of rules of thumb that
can be helpful to consider when assembling applications and I thought I'd write some of them down.

<!--more-->

### Infrastructure

The nature of the work I do is usually such that there aren't generally huge scaling concerns that
limit tech choices. As such I generally favour straightforward applications running in AWS Fargate
using Postgresql for storage. I'm not particularly fond of Fargate or Docker, it is just what I happen
to have ended up using.

I often find people choosing tools because they scale, even when the application is very unlikely to
require the scale the tools can offer. This isn't inherently a bad thing - scaling is good! - but
it also isn't free. DynamoDB for example scales amazingly but you pay an enormous day-to-day cost for that,
having to very carefully decide how you structure your data based around how it will be fetched.

Microservices introduce significant difficulties around debugging, resiliance and local development so I aim for
medium-sized services, splitting applications up only if their domains are distinct enough that they're
updated for different reasons and deployed independently. Applications should almost never share databases.

### Choosing what data to store

When I'm deciding what to store in a database I find it helpful to think about what
questions my future self is likely to return to the database with, particularly in response to
problems. As such I always try to ensure that it is easy to find out:

- What happened
- When it happened
- Why it happened / who did it

This may seem pretty obvious, but it is surprisingly easy end up with data structured such that
determining these things becomes impossible. 

Any mutable rows in a relational database inherently throw away their old state when updated 
and this can leave you with little information about what the state of something was before it changed. 
I tend to try to limit edits to data in databases for this reason, preferring to maintain something
closer to an immutable log of changes. Whether this is worth the complexity cost though depends on the data.

Keeping a log of changes doesn't solve every problem.  All too often I've had to investigate bugs 
where it is painfully obvious _what_ happened to some data but completely impossible to determine _why_. 
It is sometimes possible to guess the source of a change from when it happened but it is always worth 
having any background tasks that modify data identify themselves in a column when they do it.

Similarly if you're offering a public API it goes without saying that keeping track of which users
added or edited which data items is crucial.

Similarly giving every row a created_at
field is a small price to pay for being able to know exactly _when_ a row appeared.


### Beware time limits

Applications within a distributed system often communicate with each other using some
kind of message queue or event bus, for example Amazon SQS or Kinesis. A dead letter queue
is a common pattern with these systems but I find that the implications of doing this
are not well thought through and can often turn small problems into big ones.

The idea goes that if part of your application fails to consume a message from the queue
then that message is put to one side on dead letter queue so that subsequent messages
can continue to be consumed. At a later stage the message can be moved out of the dead letter 
queue and back onto the normal queue for re-consumption.

This is fine _in theory_ but in my experience it is rare to come across a situation where
there's some kind of transient problem that only affects one message. It is far more common
for an application to have a total outage - perhaps due to a downstream API being unavailable - 
that results in every single message ending up on the dead letter queue.

It is also very brave to trust that an application that has unexpectedly started failing to process
_some_ messages will be behaving correctly with those that it _is_ able to process. Often failures
can result due to something changing upstream and this could mean that while some messages happen
to have a compatible structure to those the application expects there's no garantee the semantics
of the data are the same.

Dead letter queues generally don't have infinite retention and the result of this is a time limit
during which you must fix whatever the underlying issue is (and perhaps it is an issue with a third party API)
before you'll begin to experience data loss. These retention periods can be configured to be quite
long but often default to short periods of time and even if your application is being well monitored 
and maintained _now_ it is very unlikely that in five years it will be, even if it is still running.

Even assuming that the issue resulting in a dead letter is resolved in time and only affected
a small number of messages there are still potential problems lurking when the dead letters
are moved back onto the original queue as they're now going to be consumed out of order
relative to where they originally ocurred. This problem is surmountable with things like 
sequence numbers and timestamps but these have to be designed for in advance and often aren't.

The purpose of this rant is not to say that dead letter queues are bad or shouldn't be used,
I just think that they shouldn't be used _lightly_. Quite often they're set up as a catch-all
error handling solution without much consideration and a poorly configured dead letter queue can be
a huge hinderence.

### Beware distributed transactions

A surprisingly large amount of the time I spend writing software is spent managing 
[distributed transactions](https://en.wikipedia.org/wiki/Distributed_transaction). They pop up absolutely 
everywhere.

### Build for neglect

Nothing lasts forever and particularly these days it seems every bit of software is about two resignations
or six months away from being labelled as legacy and shunned forever more. A sensible company
will mitigate this by good engineering practices but even sensible companies are usually two
resignations and six months away from becoming companies that aren't very sensible at all.

### Think of the questions you're going to ask


### Database schemas

I try to keep database tables as immutable as possible. My number one priority is to make it easy to ascertain what
the state of the database was at any given point in time as that is often an important first step to take
when trying to reproduce a bug. I have many painful memories of trying to work out what must've happened
to mutable database tables based on the values of `updated_at` columns and some dead reckoning.

Other than GDPR erasures I absolutely never hard delete rows from databases, instead I'll add a nullable
`deleted_at` timestamp column which can used for soft deletions. Recording the time that the deletion occurred means
that it is possible to write queries that will return the state of the data at a given time in history.

### Outbound calls

Wherever possible I'll keep all the data the application needs in Postgres to avoid having to make outbound API calls to respond to requests. This usually means that any APIs can be very fast, responding in just a few milliseconds. In the past I've maintained services that
had to make six or more API calls and so would take seconds to respond. This approach also means that the uptime of the
application is not constrained to the uptime of any third party services.

Local stores of all the information the application needs are ideally built up by consuming messages
that are published to some kind of message bus like AWS Kinesis or Kafka.

Any writes made to an external service (including something like Kafka) 
should usually be staged in the local database and then periodically relayed to the service in the background.
I've seen this called the [transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html). Since the database should be immutable it should be safe
to use a foreign key to refer to what should be written to the third party service, rather than duplicating the 
data in an outbox table.

This approach means that if the external service is unavailable no data will be lost. Doing this has allowed
the team I'm on to serenely continue feature development as others scramble to recover lost data 
from Kafka outages.