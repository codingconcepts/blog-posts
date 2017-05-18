# Know when goroutines will finish

This example demonstrates the importance of knowing when your goroutines will finish.

It functions as a simple bank account, with a channel for crediting and another for debiting.

## Things to note

The number of workers exacerbates the problem but owing to the nature of `select` statements and the way it synchronises channel reads/writes, it's not the root cause of the issue.  You could drop the worker count to 1 and the problem will persist.

The `select` statement definitely sychronises the access to the `balance` variable.  Even if you perform all modifications to balance in synchronised blocks (e.g. `atomic.Add`), the problem will persist.