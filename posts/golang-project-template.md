# Setting up a Go project

Having coded our first production-bound system in Go, we wanted to share the experience and the things we've learned.

## Source control

svn-ignore

## Developmet environment

### IDEs

There are many decent IDEs (hyped-up text editors) for Go development.  Among the best so far, are:

* VSCode - install VSCode and then install the "Go" extension
* Atom - install Atom and then install the "go-plus" plugin

As a team, we decided to use VSCode because of our familiarity with Visual Studio and haven't looked back.  To get the best out of it, we had to set one golden rule:

**Always open your project at its root level**

So if your directory structure looks like this:

C:\work\

visual studio code -> launch.json

### Directory structure

cmd, pkg, test, workspaceRoot, vendor

## Recommended packages

## Managing dependencies

`govendor` is a Go `vendoring` tool that is aware when a dependency of yours, has dependencies of its own (sounds like a pre-requisite of such a dependency manager but sadly, this isn't the case).  This makes it very easy to use and elegantly handles the fact that people vendor their library code, even though the Go community consider it to be a **cardinal sin**.

Hence, we use `govendor` and - until `dep` is officially released - we recommend you use that to vendor your dependencies.



## Testing

### Testing web applications
### Benchmarking
### Test coverage
### Mocking third-party code
### Load testing

## Connecting to the cloud
### AWS credentials
### AWS config