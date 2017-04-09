I colleague of mine was recently writing a load-balancer for an API gateway he's writing and it proved to be a great candidate for Go's benchmarking test suite.

We both wrote an implementation for a simple round-robin algorithm and ran it through benchmark tests and the results enabled us to iterate towards an algorithm that **A)** made sense to both of us and **B)** ran quickly and efficiently.

