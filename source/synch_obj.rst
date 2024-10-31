Synchronisation Objects
=======================

Introduction
------------

Overview
--------

RV/UX features a concept of synchronisation objects that unifies various types
of synchronization primitives, namely events, semaphores, mutexes, and callouts
(timers). At its core, it allows threads to wait for one or more synchronization
objects to enter a "ready" state. Each synchronization object maintains a ready
count and a queue of waiting threads. When an object's ready count becomes
positive, it can satisfy waiting threads according to its specific acquisition
semantics, which may have side-effects on the object. Waiting for readiness and
acquisition of the object are simultaneous operations.

There are several advantages to this approach:

* A consistent interface to different types of synchronization primitives.
* Ability to wait on multiple objects simultaneously.
* Fine-grained locking for scalability.
* One-and-only-one acquisition guarantee.
* All waits can be time-limited or even simple polls.

Synchronization Objects
-----------------------

Events
~~~~~~

Events are the simplest synchronization objects. When an event's ready count is
1, it satisfies waiting threads without consuming the ready state. This makes
events useful for notification scenarios where multiple waiters should be
awakened by a single signal.

.. code-block:: c

    /* Example event usage */
    struct event evt;
    event_init(&evt);

    /* Sets ready count to 1 */
    event_signal(&evt);

    /* Both waits return immediately */
    synch_wait1(&evt, "wait1", true, ABSTIME_FOREVER);
    synch_wait1(&evt, "wait2", true, ABSTIME_FOREVER);

As an example, events are used to publish the state of the physical memory
manager. There are events that are signalled on various conditions, such as
when there are lots of free pages available.

Semaphores
~~~~~~~~~~

Semaphores maintain a ready count that can vary from 0 to any positive number
under about 4 billion. Each successful wait operation has the side-effect of
decrementing the ready count, providing semantics which are useful for
representing a finite resource:

.. code-block:: c

    /* Semaphore managing 3 resources */
    struct semaphore sem;
    semaphore_init(&sem, 3);    /* Initial ready count = 3 */

    synch_wait1(&sem, "wait1", true, ABSTIME_FOREVER);  /* Count = 2 */
    synch_wait1(&sem, "wait2", true, ABSTIME_FOREVER);  /* Count = 1 */
    synch_wait1(&sem, "wait3", true, ABSTIME_FOREVER);  /* Count = 0 */
    /* Next wait blocks until count increases */
    synch_wait1(&sem, "wait4", true, ABSTIME_FOREVER);

Mutexes
~~~~~~~

Mutexes are distinctive in providing ownership semantics. A mutex's ready count
is either 0 or 1, and when the count is 1, acquisition is possible. Acquisition
establishes ownership of the mutex:

.. code-block:: c

    struct mutex mtx;
    mutex_init(&mtx);

    /* When unlocked: ready count = 1, owner = NULL */

    synch_wait1(&mtx, "wait1", true, ABSTIME_FOREVER);
    /* When locked: ready count = 0, owner = acquiring thread */

    mutex_release(&mtx);
    /* Ready count = 1, owner = NULL again. */

Callouts (Timers)
~~~~~~~~~~~~~~~~~

RV/UX treats timeouts as special synchronization objects called callouts. This
unified treatment allows a timeout to be awaited in the same fashion as other
synchronisation objects. A deadline is set, and when this is reached, the
callout's ready count becomes 1 and remains so until the callout is reset:

.. code-block:: c

    struct callout co;
    callout_init(&timeout);

    /* Set timeout to 1 second from now */
    callout_set(&timeout, time_now() + NS_PER_S);

    /* Wait for the timeout to be signalled */
    synch_wait1(&callout, "event_or_timeout", true, ABSTIME_FOREVER);

    /* Until reset, further waits on callout will return immediately */

Multi-Object Wait Operations
-----------------------------

RV/UX allows threads to wait on multiple objects simultaneously, waiting until
one of the objects can be acquired. This capability is useful for some scenarios
where there may be several conditions that should cause a thread to take action:

.. code-block:: c

    void *objects[3];
    objects[0] = &mutex;
    objects[1] = &event;
    objects[2] = &semaphore;

    /* Wait for any object to become ready */
    result = synch_waitn(3, objects, "multi_wait", true, ABSTIME_FOREVER);

    /* Poll objects without blocking */
    result = synch_waitn(2, objects, "poll", true, ABSTIME_NEVER);

    /* Wait with timeout */
    result = synch_waitn(2, objects, "timed_wait", true, deadline);

For example, the balance set scheduler uses multi-object waits to wait until
either a periodic 1-second timer or the event indicating low number of available
pages to become signalled:

Role
----

The object synchronization framework is a core component of RV/UX, and is at the
basis of most synchronisation in the system.
The framework has several properties which are useful in its capacity as the basis
of other mechanisms:


1. **Guaranteed Single Acquisition**: The framework guarantees that only one
   thread will acquire an object when it becomes ready, even if multiple threads
   are waiting.

2. **Extensibility**: New synchronization primitives can be easily added by
   implementing their acquisition semantics.

3. **Unity**: The framework provides a unified way to wait on diverse
   synchronisation primitives, including timers, without special cases.

Higher-level facilities such as the `rwlock` reader-writer lock are implemented
in terms of the system, and provide more sophisticated semantics as well as
the optimisations made possible by being specific implementations.

Implementation Details
----------------------

Wait States
~~~~~~~~~~~

The synchronization mechanism uses atomic state transitions to coordinate
between waiters and signalers. Each thread maintains a synchronization state:

.. code-block:: c

    enum synch_status {
        SYNCH_PRE_WAIT,    /* Thread is setting up wait blocks */
        SYNCH_WAIT,        /* Thread is committed to waiting */
        SYNCH_POST_WAIT    /* Wait has been satisfied */
    };

Wait Block
~~~~~~~~~~

These are the  heart of the synchronization mechanism.
Each wait block represents one thread waiting on one object, associating waiters
with objects.

Each thread preparing to wait allocates an array of wait blocks - one for each
object it will wait on. To simplify the common case, threads have 4 preallocated
wait blocks which can be used for waits on fewer than 4 objects (or 3, if there
is a timeout, as timeouts involve a hidden extra wait block.) The wait blocks
are organised as an array.

Meanwhile, each object maintains a wait queue. This is organised as a tail queue
(a doubly-linked list with head and tail pointers) of wait blocks.

Suppose we have three objects, a, b, and c, and three threads, 1, 2, and 3.
Thread 1 waits on objects a, b, and c, thread 2 waits on a and b, and thread 3
waits on a and c. Thread 1 was the first to perform a wait operation, followed
by thread 2, then thread 3.

The wait blocks are organised into a matrix-like structure where:

* Each row corresponds to a thread and its array of wait blocks.
* Each column corresponds to an object and its wait queue.
* Each cell corresponds to a wait block, tying waiting thread with object.

::

                  Object a         Object b           Object c
                     │                 │                 │
                     ▼                 ▼                 ▼
   Thread 1 ──── [ WB 1a ] ─────── [ WB 1b ] ─────── [ WB 1c ]
                     │                 │                 │
                     ▼                 ▼                 │
   Thread 2 ──── [ WB 2a ] ─────── [ WB 2b ]             │
                     │                                   │
                     ▼                                   ▼
   Thread 3 ──── [ WB 3a ] ───────────────────────── [ WB 3c ]

The wait blocks are linked into the object's wait queue in the order that waits
were initiated on the object, and when the object becomes ready, it starts
satisfying waiters from the head of their queue.

Each wait block tracks its own status to coordinate between thread and object:

.. code-block:: c

    enum waitblock_status {
        WAITBLOCK_ACTIVE,    /* Block is in object's wait queue */
        WAITBLOCK_INACTIVE,  /* Block was removed without acquisition */
        WAITBLOCK_ACQUIRED   /* Block was removed and object satisfied the wait */
    };

Wait Procedure
--------------

The wait operation proceeds in three distinct phases:

Preparation Phase
~~~~~~~~~~~~~~~~~

During preparation, the thread registers its interest in one or more objects by
enqueuing wait blocks in the objects' wait queues. If an object is already
ready, then the thread attempts early satisfaction by transitioning to the
SYNCH_POST_WAIT state.

During this phase, an object which becomes ready after a wait block has been
appended to it can also satisfy the wait early.

.. code-block:: c

    /* Initialize thread state */
    thread->synch_status = SYNCH_PRE_WAIT;

    /* For each object to wait on... */
    for (each object in wait set) {
        spin_lock(object->lock);

        if (object->ready_count > 0) {
            /* Try early satisfaction */
            if (CAS(thread->synch_status,
                   SYNCH_PRE_WAIT, SYNCH_POST_WAIT)) {
                satisfier = object;
                acquire_object(object, thread);
                spin_unlock(object->lock);
                break;
            }
        }

        /* Register wait block */
        wait_block->status = WAITBLOCK_ACTIVE;
        wait_block->thread = thread;
        queue_insert(object->waitq, wait_block);

        spin_unlock(object->lock);
    }

Commit Phase
~~~~~~~~~~~~

If no object was ready during preparation, the thread attempts to sleep:

.. code-block:: c

    spin_lock(thread->lock);

    if (CAS(thread->synch_status,
           SYNCH_PRE_WAIT, SYNCH_WAIT)) {
        thread->state = THREAD_SLEEPING;
        schedule_away();  /* Returns when thread is awakened */
    } else {
        /* Early wait satisfaction occurred. */
    }

    spin_unlock(thread->lock);

It may be that early wait satisfaction happened before the thread could begin to
sleep. In this case, the sleep is skipped and the thread proceeds to the
completion phase.

Completion Phase
~~~~~~~~~~~~~~~~

After waking (or early wait satisfaction), the thread cleans up its wait blocks
and determines which object satisfied the wait:

.. code-block:: c

    /* For each object we prepared to wait on... */
    for (each object in wait_set) {
        wit_block = wait_blocks[object];

        spin_lock(object->lock);

        switch (wait_block->status) {
        case WAITBLOCK_ACTIVE:
            /* Remove from wait queue */
            queue_remove(object->waitq, wait_block);
            break;

        case WAITBLOCK_ACQUIRED:
            /* This object satisfied our wait. Already removed by signaller. */
            satisfier = object;
            break;

        case WAITBLOCK_INACTIVE:
            /* Already removed by signaler */
            break;
        }

        spin_unlock(object->lock);
    }

Timeouts are handled by adding a hidden callout object to the thread's wait
set; if the satisfying wait block belongs to the hidden callout object, then the
wait has timed out.

Signaling Implementation
------------------------

The signaling process must handle both sleeping and preparing threads. In
C-like pseudo-code:

.. code-block:: c

    signal_object(object) {
        queue_head(wait_block) wake_q;

        spin_lock(object->lock);

        while (object->ready_count > 0 && !queue_empty(object->waitq)) {
            wait_block = queue_first(object->waitq);
            thread = wait_block->thread;

            /* Try to satisfy a preparing thread */
            if (CAS(thread->synch_status,
                   SYNCH_PRE_WAIT, SYNCH_POST_WAIT)) {
                wait_block->status = WAITBLOCK_ACQUIRED;
                acquire_object(object, thread);
                queue_remove(object->waitq, wait_block);
                continue;
            }

            /* Try to satisfy a sleeping thread */
            if (CAS(thread->synch_status,
                   SYNCH_WAIT, SYNCH_POST_WAIT)) {
                wait_block->status = WAITBLOCK_ACQUIRED;
                acquire_object(object, thread);
                queue_remove(object->waitq, wait_block);
                /* insert on wake_q for wakeup */
                queue_insert(wake_q, wait_block);
                continue;
            }

            /* Thread was already satisfied */
            wait_block->status = WAITBLOCK_INACTIVE;
            queue_remove(object->waitq, wait_block);
        }

        spin_unlock(object->lock);
    }

Finally, with the object lock released, the threads that were satisfied from the
``SYNCH_WAIT`` state are awakened.

Race Prevention
---------------

The implementation prevents several types of races:

Lost Wakeups
~~~~~~~~~~~~

The state machine prevents lost wakeups by ensuring a thread can't transition to
sleeping if it has been satisfied:

1. If a signal arrives before the thread attempts to sleep, the CAS to
   ``SYNCH_WAIT`` will fail
2. If a signal arrives after the thread sleeps, the signaler will wake the
   thread. The thread lock protects against it being woken until it is truly
   asleep.

Spurious Wakeups
~~~~~~~~~~~~~~~~

The synchroniation mechanism cannot have spurious wakeups because:

1. Threads only wake when definitively satisfied by an object
2. The WAITBLOCK_ACQUIRED status identifies exactly which object satisfied the
   wait
3. Only one object can satisfy a wait
4. Timeout handling uses the same mechanism as normal satisfaction
