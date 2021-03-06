---
categories:
- performance
- development
date: 2019-02-17T01:00:00Z
title: Checkpointing Failure
slug: checkpointing-failure
---

The conversation goes something like this:

> Them: "Our service can't be autoscaled, run on spot instances, or have its host restarted at random because it runs long-running tasks."

> Me: "Are the tasks idempotent?"

> Them: "No, but they're checkpointed."

> Me: "Even if we don't autoscale, run on spot instances, or ever update the host, the host can randomly fail at any time."

> Them: "Yes, but that's less often so it's ok. Throughput is ok if failure happens rarely."

> Me: "But you have a bug if the tasks can't be safely retried."

> Them: "I told you, we checkpoint it."

If you have lots of experience with batch workloads, there's probably nothing new here for you. But I had three similar conversations about this problem recently, so let's look into it.

The defining characteristic of the kinds of tasks we're talking about here is that they modify external state: they reserve a table at restaurant, they update the follower count in your social media network, they cause your book order to be shipped. These tasks are typically created by publishing to a queue which our workers are consuming, or they are generated on a schedule via something like cron.

There are two primary attributes we're concerned with here. Tasks must be **correct** and they must have acceptable **throughput.**

By correctness, we mean that the task gets the right answer and does the right work. But because these tasks modify state, correctness also implies **idempotency**. That is, if we have to retry them because the task fails for reasons out of our control, it should be safe to do so. We should not, for example, cause two of the same book to be shipped to you.

By throughput, we mean the performance of the task. Specifically in this case the number of tasks that can be processed by the worker. Tasks can vary quite a bit in how long they take, but if we have failures which cause us to start over, our throughput goes down. To reduce the amount of throughput lost, we can rely on **checkpointing**: we save our work in the middle of the job, allowing us to pick up where we left off with only the work between checkpoints lost.

The external force on these two values is the **error rate**. This is how often a task fails, for any reason. Even if the developer never writes a bug, perhaps the task has a network timeout. Perhaps the infrastructure team is making a kernel update and restarts the host. Perhaps the Kubernetes cluster reschedules the job. Or perhaps an electrical fire burns down the rack of hosts, sparing them the indignity of running Kubernetes.

In the conversation I had above, the developer is conflating the purpose of the two knobs of idempotency and checkpointing. A developer can tune the throughput of their tasks by adjusting the length of steps taken between checkpoints relative to the rate of unexpected errors. But increasing the rate of checkpoints does nothing for correctness.

**And as we'll see below, increasing the rate of checkpoints can very easily damage correctness.**

I've worked up a simple model to demonstrate the effect the two knobs of idempotency and checkpoint rate have on both correctness and throughput, at various error rates. This model ignores concurrency for clarity, but concurrent tasks make the correctness problem even more important to solve. You can follow along with the code [on GitHub](https://github.com/tgross/blog.0x74696d.com/blob/trunk/static/_code/checkpointing/checkpoint.py).

We run each set of parameters through our model for 100,000 "ticks". For each tick through our model, our task updates a pair of counters in a SQL database. In the middle of doing so, there is a small chance that the update fails. Each model reports the values for each counter. The difference in value between the two counters (if any) we'll refer to as the **drift** and it reflects correctness. The maximum value of the counter reflects the throughput. In a perfect world where there is a 0% error rate, both counters will have a value of 100,000.

Let's look at our idempotent task processor first.

```python
def idempotent_task(conn, checkpoint_steps, err_rate, tick, event_id):
    try:
        cur = conn.cursor()
        cur.execute("INSERT OR REPLACE INTO counterA VALUES (?)", (event_id,))
        maybe_error(err_rate)
        cur.execute("INSERT OR REPLACE INTO counterB VALUES (?)", (event_id,))
        maybe_checkpoint(conn, tick, checkpoint_steps)
        event_id += 1
    except Exception:
        conn.rollback()
    return event_id
```

We pass the `event_id` into the task and increment it upon success. The `event_id` is returned whether or not it has been incremented, so the next iteration will retry failed events. Additionally, we update both counters in a single **atomic transaction** so that we can't have partial updates. Note that atomicity and idempotency aren't the same thing! But you can't have idempotency without atomicity if you make multiple updates in a given task.

An alternative to retrying events would be to simply drop work that fails and not retry it. If our interest in the event is bound by time, this might be correct behavior. For example, if the event was a location update of our moving rideshare car, we might decide to ignore a stale update in favor of simply waiting for the next one.

Now let's take a look at our non-idempotent processor.

```python
def non_idempotent_task(conn, checkpoint_steps, err_rate, tick, event_id):
    try:
        cur = conn.cursor()
        cur.execute("INSERT OR REPLACE INTO counterA VALUES (?)", (event_id,))
        maybe_checkpoint(conn, tick, checkpoint_steps)
        maybe_error(err_rate)
        cur.execute("INSERT OR REPLACE INTO counterB VALUES (?)", (event_id,))
        maybe_checkpoint(conn, tick, checkpoint_steps)
        event_id += 1
    except Exception:
        conn.rollback()
    return event_id
```

This non-idempotent task represents a common source of bugs. We've tried to make it idempotent by using the `event_id` as we did in our previous task. But because this isn't an atomic transaction, each table can see a different set of events! The most common way this happens in my experience is **write skew**: an application that reads from the database, and then writes values back based on those values without taking into account concurrent updaters.

I've run these two tasks with error rates ranging up to 2%. That rate is perhaps pathological, but consider a task with a 20-minute long step between checkpoints. If its host is restarted once per week for kernel updates that's a 2% "failure rate" per host, assuming nothing else goes wrong. The other parameter is checkpoint steps ranging from 1 (checkpoint every tick) to 11 (checkpoint every 11 ticks).

![diagram](/images/20190217/plot.png)

The top graph measures throughput. We can see that as the error rate increases, the throughput decreases as we'd expect. We can also see that as the frequency of checkpointing goes up, the throughput goes up. For non-idempotent tasks that checkpoint after every step, we can reach very nearly 100,000. But for each pair of idempotent and non-idempotent tasks at each value of the checkpoint steps parameter, we see that the idempotent tasks fare worse in throughput performance.

The bottom graph measures correctness. At the bottom we see a single dotted line representing all the idempotent tasks together: they have no drift between the counters! But for non-idempotent tasks, we can see that as they checkpoint more frequently, not only does the checkpointing not help their correctness, but it compounds the errors they make.

What this demonstrates is that correctness cannot be truly solved by improving the failure rate of your infrastructure. If you want the wrong answer quickly, feel free to checkpoint without idempotency. But if you want software that works, your tasks need to be idempotent.
