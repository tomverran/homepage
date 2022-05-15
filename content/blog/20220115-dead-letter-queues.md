---
title: The dangers of dead letter queues
subtitle: Old man yells about cloud
date: 2022-01-15
---
Applications within a distributed system often communicate with each other using some
kind of message queue or event bus, for example Amazon SQS or Kinesis. The spookily named
_dead letter queue_ is a common pattern used with these systems to handle errors but
if they're not carefully configured and well thought through they can cause a lot of trouble.

<!--more-->

#### The theory of dead letter queues

The idea goes that if your application fails to consume a message from the queue
then that message is put to one side onto a dead letter queue so that subsequent messages
can continue to be consumed. At a later stage the failed message can be moved out of the dead letter 
queue and back onto the normal queue for re-consumption.

#### The reality of message failures

In my experience it is rare to come across a  temporary problem that only affects _some_ of the messages 
an application consumes. It is far more common for an application to have a total outage - 
perhaps due to a downstream API being unavailable - and this can result in every single message
 piling up onto the dead letter queue. If just one message really is malformed in some way - 
 perhaps due it not being serialised properly - it isn't going to fix itself without human intervention.

It is also brave to trust that an application which has unexpectedly started failing to process
some messages will be behaving correctly with those that it _is_ able to process. Failures
are often due to something changing upstream and this can mean that while some messages happen
to have a compatible structure there's no garantee that the semantics of the data are the same. 
Applications always make assumptions about the data they've received and a failure to process 
a message can be a sign that one of those assumptions has been broken. When this happens 
the safest response might be to stop everything and investigate rather than plow on regardless.

#### Time limits and sequencing

Dead letter queues generally don't have infinite retention and the result of this is a time limit
during which you must fix whatever the underlying issue is (perhaps it is an issue with a third party
whose timelines you have little control over) before you'll begin to experience data loss. 
These retention periods can be configured to be quite long but often default to short periods of time.

Even assuming that the issue is resolved in time and only affected
a small number of messages there are still potential problems lurking when the dead letters
are put back onto the queue they came from as they will be consumed out of order
relative to the messages that didn't fail. Depending on the domain this might not be a problem 
but it should definitely be thought about ahead of time.

#### Conclusion

The purpose of this rant is not to say that dead letter queues are bad or shouldn't be used,
I just think that they shouldn't be used _lightly_. Quite often they're set up as a catch-all
error handling solution without much consideration and a poorly configured dead letter queue can be
more of a hinderence than a help.

I've been fortunate in that most of the applications I've written have had low throughput and 
timeliness requirements and so it has often been possible to adopt a strategy of stopping the world 
and sounding alarms  when coming across unprocessable messages. This approach then allows a human 
to intervene before anything gets worse - not a very scalable solution but it has saved me a few times!

With a billing application it may be better to, for example, delay billing people by a few days due
to an unforeseen issue with an upstream system rather than continuing on and
billing lots of people incorrectly before having to ask for the money back.

Regardless of which approach you take towards handling errors in asynchronous applications it is
vital to actually _try_ recovering from errors, perhaps by deliberately introducing some problems in
a preproduction environment, to ensure that you've not overlooked anything that may sabotage
attempts to recover from real production issues.