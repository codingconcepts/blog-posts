### Introduction

###### Foreword

This project is by no means complete!  If you'd like to get involved and cause some destruction, I'd love to have some contributors and pull requests!

[github.com/codingconcepts/albert](https://github.com/codingconcepts/albert)

For my latest last hackathon project, I decided to roll-my-own chaos monkey.

Why not just use the Netflix Simian Army suite I hear you cry?  The answer's simple, we don't use [Spinnaker](http://www.spinnaker.io/) for continuous delivery and that's an essential part of their chaos monkey.

### Design

Having poorly-designed enough software in my time, I know when it's less shit than it could be.  Here are my design decisions/assumptions:

###### Everything will be controlled by an `orchestrator`

* Give one component a brain and keep all other components stupid.

* The orchestrator will issue commands, it won't perform actions.

###### Every action will be performed by an `agent`

* A simple, brainless agent will do one thing and do it well.

* Go binaries are tiny, so it makes sense to have an agent everywhere something needs to happen.

###### The orchestrator won't be aware of the agents

* I don't want to have to do anything when an agent is added or removed.

* I don't want to blast holes in my infrastructure just to give a chaos monkey access to machines.

###### Agents won't be aware of the orchestrator

* See above

###### All communication will be done via a messaging

* Inherently more scalable as agents come and go.

* Prevents nodes from knowing about the environment they're operating in.

###### Agents will run for particular `application groups`

* This further abstracts the orchestrator from the agents.

* Grouping applications will allow the orchestrator to be smart about how much or how little of an application's services it affects.

* An application will be something like "StatsRabbitMQ" or "CatVideoAPIServer".

###### There'll be no "un-kill" operation

* All operations will test an application's ability to recover from failure

* All operations will test the ability of interconnected applications to recover

### Technologies used

###### Messaging

I decided to bake [NATS](http://nats.io/) into my solution for the first cut.  It's very simple, easy to configure and cluster and the [Go client](https://github.com/nats-io/go-nats) is brilliant.  NATS is such a good fit for this type of project that I think I'll leave it as a baked-in messaging solution.  Woot, now I can use the increasingly popular `"{{.ProjectName}}, an opinionated {{.ProjectType}}"` project headline ;)

I'm using the [scatter-gather](http://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html) pattern, which allows the orchestrator to publish a request for agent responses indirectly:

**Orchestrator**:  "Who's responsible for managing "StatsRabbitMQ" services?"

**Agent 1 on MACHINE01**:  "I am!"

**Agent 2 on MACHINE02**:  "I am!"

**Agent 3 on MACHINE03**:  "I am!"

**Agent 4 on MACHINE04**:  "I am!"

**Orchestrator**:  "Ok, well Agent 1 and Agent 3, wherever you are, kill your service now"

**Agent 1**:  *Kills docker container called "rabbit-server" on Linux box MACHINE01*

**Agent 3**:  *Kills process called "epmd.exe" on Windows box MACHINE03*

Minus the error handling and logging for brevity, here's the scatter-gather function from the orchestrator's perpective:

``` go
func (o *Orchestrator) Process(a Application) {
    agents, _ := o.ScatterGather(a.Name)
    
    for _, agent := range model.TakeRandom(agents, a.Percentage) {
        if err := o.IssueKillCommand(agent); err != nil {
            o.Logger.Error(err)
        }
    }
}
```

Minus the error handling and logging (again for brevity), here's the scatter-gather function from the agent's perspective:

``` go
func (a *Agent) Start() {
    gatherChan, gatherStop, _ := a.chanSubscribe(application)
    defer gatherStop()

    for {
        select {
        case msg := <-gatherChan:
            a.Conn.PublishRequest(msg.Reply, a.KillInbox, []byte(application))
        // other select cases omitted
        }
    }
}
```

###### Scheduling

I'm using [cron](https://en.wikipedia.org/wiki/Cron) in the orchestrator to schedule tasks.  I decided on cron because it's familiar.  When people dive into the guts of my chaos monkey, I want them to feel at home, not like they're having to learn new concepts just to get it to work.  Each task performs a scatter-gather operation for an application group to ascertain the agents configured for that application.

Here's the orchestrator's startup scheduling code in its entirety:

``` go
func (o *Orchestrator) Start() {
    o.cronRunner = cron.New()
    for _, c := range o.Applications {
        if err := o.cronRunner.AddFunc(c.Schedule, func() { o.Process(c) }); err != nil {
            o.Logger.Fatal(err)
        }
    }

    o.cronRunner.Run()
}
```

### Configuration

Following on from my initial design decisions, the orchestrator is responsible for scheduling and nothing more, while the agents are responsible for killing processes/machines etc. and nothing more.  That's made for some pretty straightforward configuration (obvious bits have been omitted):

###### Orchestrator config

``` json
{
    "gatherTimeout": "2s",
    "gatherChanSize": 10,
    
    "applications": [
        {
            "name": "notepad",
            "schedule": "1 * * * * *",
            "percentage": 0.5
        }
    ]
}
```

<table>
    <th>Key</th>
    <th>Value</th>
    <tr>
        <td>gatherTimeout</td>
        <td>How long to wait (time.Duration for scatter-gather responses before continuing).</td>
    </tr>
    <tr>
        <td>gatherChanSize</td>
        <td>A higher number allows for more agents to response in quick succession (a Go channel will block writes if the buffer isn't large enough).</td>
    </tr>
    <tr>
        <td>applications</td>
        <td>A collection of application groups.  Each contains the application's name, the kill schedule and the percentage of an application's agents that will be asked to perform a kill on each run.</td>
    </tr>
</table>

###### Agent config

``` json
{
    "application": "notepad",
    "applicationType": "process",
    "identifier": "notepad.exe",
}
```

<table>
    <th>Key</th>
    <th>Value</th>
    <tr>
        <td>application</td>
        <td>The name of the application group this agent performs actions against.  This needs to match up with the application name known by the orchestrator.</td>
    </tr>
    <tr>
        <td>applicationType</td>
        <td>This tells the agent what kind of application it needs to kill, be it a process, the machine itself or a docker container etc.</td>
    </tr>
    <tr>
        <td>identifier</td>
        <td>The process name or docker image name</td>
    </tr>
</table>