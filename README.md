gopera is an [actor](https://en.wikipedia.org/wiki/Actor_model) framework for Golang.

# Principles
gopera is based on three principles: **Configure**, **Discover**, **React**
* Configure the actor behaviour depending on given event types
* Discover the other actors automatically registered in a registry
* React on events sent by other actors

# Main features

The main features are the following:
* Manage a hierarchy of actors
* Each actor can be either local or distributed (AMQP broker)
* Send, Forward and Repeat messages between actors
* Become/Unbecome capability to modify an actor behavior at runtime
* Automatic registration and discovery of the actors (etcd registry)

You should check the following examples for more details

# Hello world

```go
package main

import (
	"github.com/teivah/gopera/gopera"
)

func main() {
	//Init a local actor system
	gopera.InitLocalActorSystem()

	//Create an actor
	parentActor := gopera.Actor{}
	//Close an actor
	defer parentActor.Close()

	//Create an actor
	childActor := gopera.Actor{}
	//Close an actor
	defer childActor.Close()
	//Register a reaction to event types ("message" in this case)
	childActor.React("message", func(message gopera.Message) {
		message.Self.LogInfo("Received %v\n", message.Data)
	})

	//Register an actor to the system
	gopera.ActorSystem().RegisterActor("parentActor", &parentActor, nil)
	//Register an actor by spawning it
	gopera.ActorSystem().SpawnActor(&parentActor, "childActor", &childActor, nil)

	//Retrieve actor references
	parentActorRef, _ := gopera.ActorSystem().ActorOf("parentActor")
	childActorRef, _ := gopera.ActorSystem().ActorOf("childActor")

	//Send a message from one actor to another (from parentActor to childActor)
	childActorRef.Send("message", "Hi! How are you?", parentActorRef)
}
```

```
INFO: [childActor] 1988/01/08 01:00:00 Received Hi! How are you?
```

# Examples

## Message forwarding

```go
//Create an actor
actor := gopera.Actor{}
defer actor.Close()
//Configure it to react on someMessage
actor.React("someMessage", func(message gopera.Message) {
    //Forward the message to fooActor and barActor
    message.Self.Forward(message, "fooActor", "barActor")
})
```

## Stateful actor

```go
//Create a structure extending the standard gopera.Actor one
type StatefulActor struct {
	gopera.Actor
	someValue int
}

//...

//Create an actor
actor := StatefulActor{}
defer actor.Close()
//Configure it to react on someMessage
actor.React("someMessage", func(message gopera.Message) {
    //Modify the actor internal state
    if message.Data == 0 {
        actor.someValue = 1
    } else {
        actor.someValue = 0
    }
})
```

## Become/unbecome

```go
//Bar behavior
bar := func(message gopera.Message) {
    if message.Data == "foo" {
        //Unbecome to foo
        message.Self.Unbecome(message.MessageType)
    }
}

//Foo behavior
foo := func(message gopera.Message) {
    if message.Data == "bar" {
        //Become bar
        message.Self.Become(message.MessageType, bar)
    }
}

//Create an actor
actor := gopera.Actor{}
defer actor.Close()
//Configure it to react on someMessage
actor.React("someMessage", foo)
```

## Remote AMQP actor

```go
//Create an actor
actor := new(gopera.Actor).React("reply", func(message gopera.Message) {
    message.Self.LogInfo("Received %v", message.Data)
})
defer actor.Close()

//Register a remote actor listening onto a specific AMQP queue
gopera.ActorSystem().RegisterActor("actor", actor, new(gopera.ActorOptions).SetRemote(true).SetRemoteType("amqp").SetUrl("amqp://guest:guest@amqp:5672/").SetDestination("actor"))
```

## Request an actor to be closed

```go
//Create an actor
actor := new(gopera.Actor)
defer actor.Close()

//Create foo actor and register it with Autoclose set to false
foo := new(gopera.Actor)
defer foo.Close()
//Implement a specific logic if a poison pill is received
foo.React(gopera.GoperaMsgPoisonPill, func(message gopera.Message) {
    //Do something
})
gopera.ActorSystem().RegisterActor("foo", foo, new(gopera.ActorOptions).SetAutoclose(false))

//Create foo actor and register it with Autoclose set to true
bar := new(gopera.Actor)
defer bar.Close()
gopera.ActorSystem().RegisterActor("bar", bar, new(gopera.ActorOptions).SetAutoclose(true))

//Retrieve the actor references
actorRef, _ := gopera.ActorSystem().ActorOf("actor")
fooRef, _ := gopera.ActorSystem().ActorOf("foo")
barRef, _ := gopera.ActorSystem().ActorOf("bar")
//Request to close foo from requester by sending a poison pill
fooRef.AskForClose(actorRef)
barRef.AskForClose(actorRef)
```

## Repeat
```go
//Retrieve the actor references
fooRef, _ := gopera.ActorSystem().ActorOf("fooActor")
barRef, _ := gopera.ActorSystem().ActorOf("barActor")

//Repeat a message every 5 ms
c, _ := fooRef.Repeat("message", 5*time.Millisecond, "Hi! How are you?", barRef)

time.Sleep(21 * time.Millisecond)

//Ask the actor system to stop the repeated message
gopera.ActorSystem().Stop(c)
```

# Contributing

* Open an issue if you want a new feature or if you spotted a bug
* Feel free to propose pull requests

Any contribution is more than welcome! In the meantime, if we want to discuss about gopera you can contact me [@teivah](https://twitter.com/teivah).