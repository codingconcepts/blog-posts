##### Minimum Viable Product
Everyone's heard of `Minimum Viable Product`.  It's the minimum amount of software you can deliver to an early adopter such that it works and they're happy with it.

My team take this a step further and will only deliver our MVP when the most important prerequisites (things it'd be irresponsible to omit) are also finished.  We still call this MVP but the V stands for `valuable`.

##### Minimum Valuable Product
The minumum fully-tested, dev-paired, dev/test-paired and prerequisite-satisfied software you can deliever to any customer such that it works and they're happy with it.

When starting a new `epic`, my team perform story-mapping to break it down into manageable `stories`.  These stories are thrashed out in a planning session and then again during a deeper `3 amigos` session.  At any stage, before or during development, we can break a story into smaller, more manageable stories or tasks.

For all stories, no matter the size, we run through a checklist which ensures we've thought about the major prerequisites.  These prerequisites include but are not limited to the following.  All important enough that we won't consider our story complete until they're satisfied:

* Unit, integration and performance tests have been written/performed
* We're writing logs and are able to view them in a decentralised place
* We're publishing stats and able to monitor them
* A circuit-breaker is in place if our software relies on a potentially unreliable third-party service
* Tech/product/support documentation has been written

Unfortunately, I've recently observed a growing trend towards BVP, a term coined just today by my team's excellent Product Owner.  It stands for `Barely Viable Product` and we're already over-using it.

##### Barely Viable Product
The minimum, casually untested software deemed by its solo developer as being *probably ok* for production.  Our customers will be happy with it if it works but will let us know if they're not happy because it's not working.

This comes in multiple flavours and they all taste foul.  It seems to result from a combination of the following:

* [Failure Demand](https://en.wikipedia.org/wiki/Failure_demand)
* Over-bearing product team influence
* Developing in silos
* Refusal to see the value of peer reviews
* Developers not taking responsibility / being held accountable for the software they develop
* Bad ratio of developers to testers (1 tester trying to test a wave of disparate developments from multiple developers)
* Entirely untested changes being considered low-risk for production

The result of a BVP can also come in multiple flavours:

* More time spent fire-fighting
* More time spent yak-shaving around technical dept
* More support calls from unhappy customers (as they might - shamefully - be the first testers of the system)
* An increased average number of weekly stop-the-lines
* An increased number of 5-Whys sessions