# A modular cluster controller for burst autoscaling and graceful reconfiguration

We recently added a bundle of new cluster features: burst autoscaling and
graceful reconfiguration. I will explain these below by way of example, but to
me the more exciting thing is that we managed to add these _and_ improve how
our control plane is architected. Which latter I will explain after we first
explore the examples.

## Burst Autoscaling

Say we have an existing big collection `burst_demo_events` and we want to
create a new index on it. This requires creating a new dataflow which can be
quite a while to hydrate.

To reduce the time we have to wait, we can now specify an auto scaling strategy
on the cluster that will spin up a large _burst_ replica while the regular
_steady state_ replica is hydrating.

As SQL:

```sql
CREATE CLUSTER burst_demo (SIZE = '100cc');

ALTER CLUSTER burst_demo SET (
    AUTO SCALING STRATEGY = (
        ON HYDRATION (
            HYDRATION SIZE = '800cc',
            LINGER DURATION = '60s'
        )
    )
);

CREATE INDEX burst_demo_events_idx
    IN CLUSTER burst_demo ON burst_demo_events (id);
```

We can check in on the cluster to observe the burst while it happens:

```sql
SELECT name, replicas, activity
FROM (SHOW CLUSTERS)
WHERE name = 'burst_demo';
\watch 1
```

Example output, during the hydration burst:

```
    name    |         replicas         |         activity
------------+--------------------------+---------------------------
 burst_demo | r1 (100cc), r2 (800cc)    | hydration burst at 800cc
(1 row)
```

And, once the steady state replica is done hydrating and the burst is retired, we get:

```
    name    |  replicas  | activity
------------+------------+----------
 burst_demo | r1 (100cc) |
(1 row)
```

You can read up on the details of burst autoscaling strategies in our docs: [Autoscaling for hydration](https://materialize.com/docs/clusters/autoscaling/).

## Graceful (Background) Cluster Reconfiguration

For a while, we've had a feature that allows reconfiguring a cluster without
causing downtime: graceful cluster reconfiguration. When issuing an `ALTER
CLUSTER [...]` statement that, say, changes the size, we would spin up a
replica of the new size, keep the old replica, and only retire the old-size
replica when the new-size replica is hydrated. This worked, but had the
limitation that you had to keep the SQL client session open while
reconfiguration was happening.

As part of the burst autoscaling work, we took the opportunity to improve our
code and factor out all cluster controlling machinery into a shared _cluster
controller_. And as a "side effect", we also made it so that graceful
reconfiguration now works in the background, without requiring an active SQL
client connection.

As a user you can now use it like the following. Keep querying in one psql session:

```sql
SET cluster = burst_demo;
SELECT payload FROM burst_demo_events WHERE id = 42;
\watch 1
```

In another session, resize and observe:

```sql
ALTER CLUSTER burst_demo SET (SIZE = '200cc');

SELECT name, replicas, activity
FROM (SHOW CLUSTERS)
WHERE name = 'burst_demo';
\watch 1
```

The `ALTER` statement will return immediately and the reconfiguration will continue in the background.

During reconfiguration, the above watch query will yield something like:

```
    name    |         replicas         |          activity
------------+--------------------------+-----------------------------
 burst_demo | r1 (100cc), r2 (200cc)    | reconfiguring size to 200cc
(1 row)
```

You can read up on cluster resizing in our docs as well: [Cluster resizing](https://materialize.com/docs/sql/alter-cluster/#resizing)

## Background: The Coordinator Loop

Simplifying quite a bit, the materialize control plane is an `environmentd`
process that drives around the `clusterd` processes, which constitute the data
plane. Within `environmentd`, there is one central control loop of what we call
the `Coordinator` that receives internal and external _commands_, processes
them, and dispatches new commands to itself or the clusters.

As pseudo Rust code, you can imagine it mostly like this:

```rust
loop {
    select! {
        cmd = internal_cmd_rx.next() {
            process_internal_cmd(cmd).await
        }
        cmd = user_cmd_rx.next() {
            process_cmd(cmd).await
        }
        [..]
    }
}
```

We inherited this design from the very first days of materialize, where using a
central sequentializing loop was the easiest approach for providing the
correctness experience (read _consistency_) of Materialize. It comes with the
downside that one central loop that has to process everything limits the amount
of work you can do, as there are only about 1000ms in a second. Over time, we
have grown many optimizations that offload work from this central loop to
background tasks and execution stages to squeeze out more performance from this
design. But it has it's limits, and one of the long-running goals of the team
working on the control plane is to make it "infinitely" scalable both
vertically and horizontally.

Before our work on the cluster controller, the existing (foreground) graceful
reconfiguration feature was implemented as messages and execution stages that
are sequenced by this central control loop. And the obvious, but in my opinion
misguided, approach for adding burst autoscaling would have been to add the
functionality right there and in similar fashion to the existing graceful
reconfiguration.

Instead, what we did is take a step back, think holistically about what the
(long-term) architecture _should_ be, and implemented that.

## Cluster Controller

The long-term vision for the control plane requires that `environmentd` (or
it's replacement processes) be both vertically and horizontally scalable.
Vertically scalable meaning that we can do more useful work when giving it more
CPU cores. So handling larger query workloads, larger numbers of objects and
clusters, etc. And horizontal scalable meaning that we can have multiple
processes coordinating and sharing the control plane workload for a given
environment.

In our work here, we focused on vertical scalability, while making sure that
the architecture will also work in a horizontal-scaling world.

The overall approach to vertical scalability is simple: extract as much work
out of the central processing loop into independent "tasks" (in this case,
`async` Rust tasks). Put in place a minimal interface/boundary between these
tasks and the controller loop.

Putting that into practice with the cluster controller meant that we had to do
roughly these things:

1. Add the SQL surface for creating auto scaling strategies and kicking of
   cluster reconfigurations.
2. Add commands and responses that the controller sends to the Coordinator.
3. Add the new cluster controller that enacts things from #1.

Previously, the flow of processing SQL commands would be handled by the
Coordinator loop, across multiple execution stages that send things to
background tasks and then require processing time on the main loop again.

Now, processing the SQL statement only requires writing down the intent in our
durable catalog, and then the cluster controller reads those in the background,
figures out what to do and sends messages to the Coordinator when needed.

TODO: generate figure for the architecture of Coordinator, controller, and the messages.

## Links

- [Design: Cluster Autoscaling and Background Reconfiguration](https://github.com/MaterializeInc/materialize/blob/main/doc/developer/design/20260522_cluster_autoscaling.md)
- [Implementation: Hydration burst autoscaling (#37671)](https://github.com/MaterializeInc/materialize/pull/37671)
- [Implementation: Background graceful reconfiguration (#37452)](https://github.com/MaterializeInc/materialize/pull/37452)
- [Implementation: Reconfiguration observability and SHOW CLUSTERS (#37628)](https://github.com/MaterializeInc/materialize/pull/37628)
- [Docs: Autoscaling for hydration](https://materialize.com/docs/clusters/autoscaling/)
- [Docs: Cluster resizing](https://materialize.com/docs/sql/alter-cluster/#resizing)
- [Docs: SHOW CLUSTERS](https://materialize.com/docs/sql/show-clusters/)

For readers interested in the shared controller architecture:

- [Implementation: Cluster controller framework (#37214)](https://github.com/MaterializeInc/materialize/pull/37214)
- [Original design discussion (#36691)](https://github.com/MaterializeInc/materialize/pull/36691)
