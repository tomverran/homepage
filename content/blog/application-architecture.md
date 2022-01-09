---
title: How I architect software in 2022
date: 2022-01-09
draft: true
---

This article is a WIP, I split it out from the Scala article to prevent it becoming too long. 

### Infrastructure

The nature of the work I do is such that there aren't generally huge scaling concerns that
limit tech choices. As such I generally favour straightforward applications running in AWS Fargate
using Postgresql for storage. I'm not particularly fond of Fargate or Docker, it is just what I happen
to have ended up using.

Microservices introduce significant difficulties around debugging, resiliance and local development so I aim for
medium-sized services, splitting applications up only if their domains are distinct enough that they're
updated for different reasons and deployed independently. Applications should almost never share databases.

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