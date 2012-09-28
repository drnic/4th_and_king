# 4th and King

When you start bringing up a new system, during its general operation and during outages of various components, you want to know what's going on. Are the individual components ok? Are the components all communicating with each other?

The faster you know what pieces are not working as expected, the faster you can diagnose the root cause and bring them back to life.

> Why "4th and King"? 

It is the name of the first Caltrain station, leaving San Francisco southbound through Silicon Valley, on to San Jose and then Gilroy.

> I mean, why call the project "4th and King"? 

It's a metaphor.

> Of what?

Of starting a journey the same way you intend to finish it.

> That's quite a stretch.

Ok, fine. I was siting on the Caltrain, at 4th and King station waiting to go home when created the project. Happy?

> This project better be good.

...

## What does it do?

* Continuously sends inert tracer events through a complex system to test that everything is working
* Continuously sends local tracer events into each component (if possible) to test that everything is working
* Can receive any other monitored data about the components from other systems
* Provides aggregate reports of the monitored system and its components
* Announces the aggregate reports to subscribers

## Example usage

The original use case was that I wanted a system that could tell me when my entire Logstash system was primed and running. It wasn't sufficient to know "when are all the processes running?" I wanted to know "if I put a sample log event into Logstash, is the dashboard working yet and will it show the sample logged event?"

Logstash is composed of several loosely coupled services. In order of a log event flowing through the system:

* Remote server, running a Logstash agent to collect new log events (can take 30s+ to boot up and start collecting events)
* A broker (Redis or AMPQ) to queue all initial events (Redis is quick to become available)
* Indexer (another Logstash agent, which can take 30s+ to start working) to take the queued events and store in the search engine
* Search engine (ElasticSearch, which can take 30s+ to start working)
* Logstash Web app (Logstash Web, which is dependent on Elastic Search running)

I want to know when the entire system is running. I also want to know if any part is misbehaving - say the Indexer - and what ill-effects it is causing on the rest of the system. For example, if the Indexer stops pulling events from Redis, then Redis will start filling up with events and may run out of memory.

When I say, "I want to know", there may be multiple systems that collect the status information of the system and its components. 

4th and King should be constantly sending inert tracers through a system and watching for the basic availability of each component.

## Architecture

```
Subscribers <---|
                |
Logfiles <-- Public API <--- DB <---- Collector -- -- --> Services
                ^
Pollers --------|
```

The collector processes will need to have access to the Services (in the Example Usage, these were Redis, Indexer, Elastic Search, Web app), and also have access to the 4th & King DB to store the data.

The modular nature of this architecture means that each part can be replaced/improved/scaled over time independently of the others. The collector will retain data locally if the DB is temporarily not available. The Public API can itself be composed of separate systems.

Initially, this implementation is:

* Public API - Sinatra for the Polling API, Sidekiq for background workers to collect from the DB and announce to the Subscribers
* DB - Redis (caching mode, as it only needs to maintain a window of recent history; want a bigger window? have a bigger Redis)
* Collector - Sidekiq background workers that each perform actions on a system or its components and watch what happens, then report back to the DB

