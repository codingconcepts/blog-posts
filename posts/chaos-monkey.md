For my company's last hackathon, I decided to roll-my-own chaos monkey.

I know, I know, I can hear the questions already...  Why not just use the Netflix Simian Army suite?  The answer's simple, we don't use Spinnaker and that's an essential part of their chaos monkey.

By far the most difficult part of this project was its design.  There are many ways to crack a nut but some just seem less shit than others (and I don't want to be caught with my pants-down when it comes to adding new features later on).

### Overview

I made some fundamental decisions/assumptions up-front.  These not only helped the design along but - in my opinion - have paved the way to the solution being more scalable and generally more sensible:

#### Everything will be controlled by an `orchestrator`

* This prevents logic from bleeding into components that could otherwise be very simple.

* Configuration can be done once and in one place.

* A simple scheduler will be used to execute actions.

* It it goes rogue, I'll have a brain to turn off.

#### An `agent` will be installed on every machine where something needs to happen

* The orchestrator will issue commands, it won't perform actions.

* I'll have a simple agent that does one thing and does it well.

* Go compiles to tiny binaries and you don't need Go to run Go, so hoofing tiny agent applications around is no problem.

* Regardless of the OS/platform, I'll be able to compile an agent for it.

#### The orchestrator will not be aware of the agents

* I don't want to have to do anything when an agent is added or removed.

* I don't want to blast holes in my infrastructure just to give a chaos monkey access to machines.

#### Agents will run for particular `applications`

* An application is something like "StatsRabbitMQ" or "TelegrafServer".

* Grouping applications will allow the orchestrator to be smart about how much or how little of an application's services it affects.

#### All communication will be done via a messaging layer

* Inherently more scalable as agents come and go.

* Offers a level of indirection, preventing all nodes from knowing more than they need to.

### Technologies used

#### Messaging

I decided to bake [NATS](http://nats.io/) into my solution for the first cut.  I've played with it before and it does enough of what a clunkier AMQP message bus like RabbitMQ would be able to do, to make my project work.  I'd definitely consider adding support for additional message buses in the future but for now, this is what NATS was born to do.

I'm making use of the [scatter-gather](http://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html) pattern to allow the orchestrator to ask for agents handling a particular application.

Minus the error handling for brevity, here's the scatter-gather function from the orchestrator's perpective:

``` go
func (o *Orchestrator) Process(a Application) {
	agents, _ := o.ScatterGather(a)

	// select a number of applications at random to kill
	randomAgents := model.TakeRandom(agents, a.Percentage)

	for _, topic := range randomAgents {
		if err := o.IssueKillCommand(topic); err != nil {
			o.Logger.Error(err)
		}
	}
}
```

Minus the error handling (again for brevity), here's the scatter-gather function from the agent's perspective:

``` go
func (a *Agent) Start() {
	gatherChan, gatherStop, _ := a.chanSubscribe(a.Application)
	defer gatherStop()

	for {
		select {
		case msg := <-gatherChan:
			a.GatherResponse(msg.Reply)
        // ommitted other select cases
		}
	}
}

func (a *Agent) GatherResponse(reply string) (err error) {
	return a.Connection.PublishRequest(reply, a.KillInbox, []byte(a.Application))
}
```

Here's an example workflow to give you an idea what I'm doing with the scatter-gather pattern:

Orchestrator:  "Who's responsible for managing "StatsRabbitMQ" services?"

Agent 1:  "I am!"

Agent 2:  "I am!"

Agent 3:  "I am!"

Agent 4:  "I am!"

Orchestrator:  "Ok, well Agent 1 and Agent 2, wherever you are, kill your service now"

Agent 1:  Kills docker container called "rabbit-server" on Linux box MACHINE01

Agent 2:  Kills process called "epmd.exe" on Windows box MACHINE2

#### Scheduling

I'm using [cron](https://en.wikipedia.org/wiki/Cron) in the orchestrator to schedule tasks.  Each task performs a scatter-gather operation for an application group to ascertain the agents configured for that application.

I decided to use cron (namely the populare Go library from [robfig](https://github.com/robfig/cron)) to make this possible, as the resulting code is clean and easy to understand.

The following code is currently the entirety of my Orchestrator's startup logic:

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

Configuration is also simple.  Following on from my initial design decisions, the orchestrator is responsible for scheduling and nothing more, while the agents are responsible for killing processes/machines etc. and nothing more

Here's the orchestrator's config file:

``` json
{
    "natsHosts": [ "nats://localhost:4222" ],
    "natsUser": "harambe",
    "natsPass": "Imk)^InzID2QM*YMWPchUZ",
    "gatherTimeout": "2s",
    "gatherChanSize": 10,

    "logLevel": "debug",
    
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
        <td>natsHosts</td>
        <td>A colletion of NATS endpoints.  You actually need only one endpoint and NATS' new service discovery will allow your cluster to dynamically grow.</td>
    </tr>
    <tr>
        <td>natsUser</td>
        <td>Self-explanatory</td>
    </tr>
    <tr>
        <td>natsPassword</td>
        <td>Self-explanatory</td>
    </tr>
    <tr>
        <td>gatherTimeout</td>
        <td>How long to wait (time.Duration for scatter-gather responses before continuing).</td>
    </tr>
    <tr>
        <td>gatherChanSize</td>
        <td>A higher number allows for more agents to response in quick succession (a Go channel will block writes if the buffer isn't large enough).</td>
    </tr>
    <tr>
        <td>logLevel</td>
        <td>Self-explanatory</td>
    </tr>
    <tr>
        <td>applications</td>
        <td>A collection of application groups.  Each contains the application's name, the kill schedule and the percentage of an application's agents that will be asked for perform a kill on each run.</td>
    </tr>
</table>

Here's the agent's config file:

``` json
{
    "natsHosts": [ "nats://localhost:4222" ],
    "natsUser": "harambe",
    "natsPass": "Imk)^InzID2QM*YMWPchUZ",
    
    "application": "notepad",
    "applicationType": "process",
    "identifier": "notepad.exe",

    "logLevel": "debug"
}
```

<table>
    <th>Ke</th>
    <th>Value</th>
    <tr>
        <td>natsHosts</td>
        <td>A colletion of NATS endpoints.  You actually need only one endpoint and NATS' new service discovery will allow your cluster to dynamically grow.</td>
    </tr>
    <tr>
        <td>natsUser</td>
        <td>Self-explanatory</td>
    </tr>
    <tr>
        <td>natsPassword</td>
        <td>Self-explanatory</td>
    </tr>
    <tr>
        <td>application</td>
        <td>The name of the application group this agent performs actions against.  This needs to match up to the application name known by the orchestrator.</td>
    </tr>
    <tr>
        <td>applicationType</td>
        <td>This tells the agent what kind of application it needs to kill, be that a process, the machine itself or a docker container etc.</td>
    </tr>
    <tr>
        <td>identifier</td>
        <td>The process name or docker image name</td>
    </tr>
    <tr>
        <td>logLevel</td>
        <td>Self-explanatory</td>
    </tr>
</table>