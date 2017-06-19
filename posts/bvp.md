##### Minimum Viable Product
Everyone's heard of `Minimum Viable Product`.  It's the minimum amount of software you can deliver to an early adopter such that it works and they're happy with it.

My team take this a step further and will only deliver our MVP when the most important prerequisites (things it'd be irresponsible to omit) are also finished.  We also call this MVP but the V stands for `valuable`.

##### Minimum Valuable Product
These important prerequisites include but are not limited to:

* Unit, integration and performance tests have been written/performed
* We're writing logs and are able to view them
* We're publishing stats and able to monitor them
* A circuit-breaker is in place if our software relies on a third-party
* Tech/product/support documentation has been written

Unfortunately, I've recently observed a growing trend towards BVP, a term coined by my team's excellent Product Owner; `Barely Viable Product`.

##### Barely Viable Product
This comes in multiple flavours and they all taste pretty foul.  It seems to result from a combination of the following:

* [Failure Demand](https://en.wikipedia.org/wiki/Failure_demand)
* Over-bearing product team influence (which results in a drive for new features at-any-cost and to the detriment of technical dept etc.)
* Management/architect oversight
* Developers working in silos
* Developers refusing to see the value of peer reviews
* Developers not taking responsibility / being held accountable for the software they develop
* Bad ratio of developers to testers (1 tester trying to test a wave of disparate developments from multiple developers)
* Entirely untested changes

The result of a BVP can also come in multiple flavours:

* More time spent fire-fighting
* More time spent yak-shaving around technical dept
* More support calls from unhappy customers (as they might - shamefully - be the first testers of the system)