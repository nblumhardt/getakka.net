---
layout: docs.hbs
title: Fault tolerance
---
# Fault Tolerance

As explained in [Actor Systems](ActorSystem) each actor is the supervisor of its
children, and as such each actor defines fault handling supervisor strategy.
This strategy cannot be changed afterwards as it is an integral part of the
actor system's structure.

## Fault Handling in Practice

First, let us look at a sample that illustrates one way to handle data store errors,
which is a typical source of failure in real world applications. Of course it depends
on the actual application what is possible to do when the data store is unavailable,
but in this sample we use a best effort re-connect approach.

Read the following source code. The inlined comments explain the different pieces of
the fault handling and why they are added. It is also highly recommended to run this
sample as it is easy to follow the log output to understand what is happening in runtime.

!!!TODO: Port sample code

## Creating a Supervisor Strategy

The following sections explain the fault handling mechanism and alternatives
in more depth.

For the sake of demonstration let us consider the following strategy:

!!!TODO: Port sample code

We have chosen a few well-known exception types in order to demonstrate the
application of the fault handling directives described in [Supervision](Supervision).
First off, it is a one-for-one strategy, meaning that each child is treated
separately (an all-for-one strategy works very similarly, the only difference
is that any decision is applied to all children of the supervisor, not only the
failing one). There are limits set on the restart frequency, namely maximum 10
restarts per minute; each of these settings could be left out, which means
that the respective limit does not apply, leaving the possibility to specify an
absolute upper limit on the restarts or to make the restarts work infinitely.
The child actor is stopped if the limit is exceeded.

This is the piece which maps child failure types to their corresponding directives.

> **Note**<br/>
If the strategy is declared inside the supervising actor (as opposed to
within a companion object) its decider has access to all internal state of
the actor in a thread-safe fashion, including obtaining a reference to the
currently failed child (available as the ``sender`` of the failure message).

### Default Supervisor Strategy

`Escalate` is used if the defined strategy doesn't cover the exception that was thrown.

When the supervisor strategy is not defined for an actor the following
exceptions are handled by default:

* `ActorInitializationException` will stop the failing child actor
* `ActorKilledException` will stop the failing child actor
* `Exception` will restart the failing child actor
* Other types of `Exception` will be escalated to parent actor

If the exception escalate all the way up to the root guardian it will handle it
in the same way as the default strategy defined above.

You can combine your own strategy with the default strategy:

!!!TODO: Port sample code

### Stopping Supervisor Strategy

Closer to the Erlang way is the strategy to just stop children when they fail
and then take corrective action in the supervisor when DeathWatch signals the
loss of the child. This strategy is also provided pre-packaged as
`SupervisorStrategy.StoppingStrategy` with an accompanying
`StoppingSupervisorStrategy` configurator to be used when you want the
`/user` guardian to apply it.

### Logging of Actor Failures

By default the `SupervisorStrategy` logs failures unless they are escalated.
Escalated failures are supposed to be handled, and potentially logged, at a level
higher in the hierarchy.

You can mute the default logging of a `SupervisorStrategy` by setting
`loggingEnabled` to `false` when instantiating it. Customized logging
can be done inside the `Decider`. Note that the reference to the currently
failed child is available as the `Sender` when the `SupervisorStrategy` is
declared inside the supervising actor.

You may also customize the logging in your own ``SupervisorStrategy`` implementation
by overriding the `logFailure` method.

## Supervision of Top-Level Actors

Top-level actors means those which are created using `system.ActorOf()`, and
they are children of the [User Guardian](User guardian). There are no
special rules applied in this case, the guardian simply applies the configured
strategy.

## Test Application

The following section shows the effects of the different directives in practice,
wherefor a test setup is needed. First off, we need a suitable supervisor:

!!!TODO: Port sample code

This supervisor will be used to create a child, with which we can experiment:

!!!TODO: Port sample code

The test is easier by using the utilities described in [Akka-Testkit](TestKit).

!!!TODO: Port sample code

Let us create actors:

!!!TODO: Port sample code

The first test shall demonstrate the ``Resume`` directive, so we try it out by
setting some non-initial state in the actor and have it fail:

!!!TODO: Port sample code

As you can see the value 42 survives the fault handling directive. Now, if we
change the failure to a more serious `NullPointerException`, that will no
longer be the case:

!!!TODO: Port sample code

And finally in case of the fatal `IllegalArgumentException` the child will be
terminated by the supervisor:

!!!TODO: Port sample code

Up to now the supervisor was completely unaffected by the child's failure,
because the directives set did handle it. In case of an ``Exception``, this is not
true anymore and the supervisor escalates the failure.

!!!TODO: Port sample code

The supervisor itself is supervised by the top-level actor provided by the
`ActorSystem`, which has the default policy to restart in case of all
`Exception` cases (with the notable exceptions of
`ActorInitializationException` and `ActorKilledException`). Since the
default directive in case of a restart is to kill all children, we expected our poor
child not to survive this failure.

In case this is not desired (which depends on the use case), we need to use a
different supervisor which overrides this behavior.

!!!TODO: Port sample code

With this parent, the child survives the escalated restart, as demonstrated in
the last test:

!!!TODO: Port sample code

## BackoffSupervisor

This actor can be used to supervise a child actor and start it again after a back-off duration if the child actor is stopped.
This is useful in situations where the re-start of the child actor should be delayed e.g. in order to give an external resource time to recover before the child actor tries contacting it again (after being restarted).

Specifically this pattern is useful for for persistent actors, which are stopped in case of persistence failures. Just restarting them immediately would probably fail again (since the data store is probably unavailable). It is better to try again after a delay.

It supports exponential back-off between the given `minBackoff` and `maxBackoff` durations. For example, if `minBackoff` is 3 seconds and `maxBackoff` 30 seconds the start attempts will be delayed with 3, 6, 12, 24, 30, 30 seconds. The exponential back-off counter is reset if the actor is not terminated within the minBackoff duration.

In addition to the calculated exponential back-off an additional random delay based the given `randomFactor` is added, e.g. 0.2 adds up to 20% delay. The reason for adding a random delay is to avoid that all failing actors hit the backend resource at the same time.

You can retrieve the current child ActorRef by sending `BackoffSupervisor.GetCurrentChild.Instance` message to this actor and it will reply with `BackoffSupervisor.CurrentChild` containing the ActorRef of the current child, if any.

The BackoffSupervisor forwards all other messages to the child, if it is currently running.

The child can stop itself and send a PoisonPill to the parent supervisor if it wants to do an intentional stop.
As long as the `BackoffSupervisor` is in the backoff state, it will deadletter any messages it would normally send to the child.

###example
```csharp
 var childProps = Props.Create(() => new MyChildActor());
 var superVisor = system.ActorOf(
  Props.Create(() =>
    new BackoffSupervisor(childProps, "child",
     TimeSpan.FromSeconds(3), //minBackoff: once it fails, retry after 3 secs
     TimeSpan.FromSeconds(30), //maxBackoff: max time between retries is 30 secs
     0.1), //random factor that influences retry times 
    new OneForOneStrategy(_ => {
        return Directive.Stop; //The BackoffSupervisor only works on the Stop Directive.
    })));
```


