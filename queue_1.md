# Fair, linearizible, blocking concurrent queue (1)

Queues are the basic building block of concurrent comunication. One group of threads/processes/workers are producing data while other group is consuming data, forming a pipeline. This is pattern is happening in every layer of modern computing going down to even the hardware level, for example individual instructions are concurrently decoded, sheduled and executed with queues in between. 

The goal of this series of blogs will be to develop an easy to use concurrent queue primitive roughly equivalent to (buffered) [Go channel](https://go101.org/article/channel.html). This type of queue has push/pop as its main operations, which add/remove item from the queue unless the queue is full/empty in which they block the calling thread untill it becomes not full/empty. Though unlike Go channels we wish to do this without locks. If the queue is not full it should be possible for two threads to push an item with minimal interference. 

> It may appear that having the block-on-full-push (implying there is finite capacity) limitation is too restrictive, dont we wish to grow for as long as we can? Well in practice no. Consider that if the production of new items is just a little faster then the consumption, the queue will eneventually fill up. Without a maximum capacity, it will keep on growing forever untill it exhausts significant portion of our memory, causing some potentially unrelated allocation to fail. This is problematic because it happens so easily - all it takes is a tiny bit slower consumer. A particular workload being slower to consume? You already have a problem. Because all of this its easier to enforce some concrete upper bound. 

After some initial research I have come across this quite excellent paper [T. R. W. Scogland - Design and Evaluation of Scalable Concurrent Queues for Many-Core Architectures, 2015](https://synergy.cs.vt.edu/pubs/papers/scogland-queues-icpe15.pdf), which has all the properties we seek. We use this paper as the basis of our design and extend it in some important ways namely allowing proper thread blocking instead of spin waiting and more useful closing behaviour. The remainder of this blog will be devoted to a hopefully intuitive explanation of the Scogland queue (queue from the paper). 

> What is fair? By fair I mean that a waiting thread cannot be starved of work. That is if thread1 calls pops from an empty queue, followed by thread2 calling pop, then (once the queue becomes nonempty) thread1 will always be allowed to run before thread2. Similar case can be made for push and full queue. This property is essential for allowing the threads to do other work while they are waiting, since the precise timing no longer matters. For example we might have a job system that while waiting starts executing some other job, yet the duration of this job has no effect on the waiting.

> The final implementation which will be reached after the entire series of blogs can be found [here](https://github.com/Boostibot/c_lib/blob/main/channel.h).

## First try

I assume the reader is more familiar with actual code then some abstract pseudocode so I will just provide a very basic version of the code from the paper. During the course of this blog we will work our way to something closer to the code in the paper. The code is written using use C11 atomics but turning it into any other language should be trivial.  

```C
#define QUEUE_CAP 10 //capacity

typedef int Item; 

typedef struct Queue {
    _Atomic uint64_t head;
    _Atomic uint64_t tail;
    _Atomic uint32_t ids[QUEUE_CAP];
            Item items[QUEUE_CAP];
} Queue;

void push(Queue* q, Item item) {
    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2;

    while(atomic_load(&q->ids[target]) != id);

    q->items[target] = item;
    atomic_store(&q->ids[target], id + 1);
}

void pop(Queue* q, Item* item_ptr) {
    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2 + 1;

    while(atomic_load(&q->ids[target]) != id);

    *item_ptr = q->items[target];
    atomic_store(&q->ids[target], id + 2*QUEUE_CAP + 1);
}

int main() {
    //initialize everything to zero
    Queue q = {0};
    push(&q, 1);
    push(&q, 2);
    push(&q, 3);
}
```

So what is this doing? To start off this queue uses virtualized head/tail indices - this is to say that head/tail only ever increase and the actual index at which the item resides (`target`) is calculated by doing a modulo with the queue capacity. If `tail > head`, the queue stores precisley `tail - head` items. If `tail == head` the queue is empty. If this is novel to you I invite you to read about this approach [here](https://fgiesen.wordpress.com/2010/12/14/ring-buffers-and-queues/). The reason why it was chosen is obvious - all we need to do to push item is a single atomic add instruction - no complicated ifs.

The queue holds an array `ids`, each of which act like a [ticket lock](https://mfukar.github.io/2017/09/08/ticketspinlock.html). On a push operation the `tail` index is increment which yields `ticket` and as discussed above `target`. We also use `ticket` to calculate `id`. This id uniquelly represents this operation: 
- the id is always (atomically) increasing so no two pushesh can have the same one (ignoring overflow) 
- ids of push operations are even (`*2`) while pop operations are odd (`*2 + 1`)

Now that we have obtained our unique `id` and some `target` we wait for the `ids[target]` to become our `id`. We for now do a simple spin loop whic is like repeatedly asking "Is it my turn? Is it my turn?", while hoepfully some other thread eventually responds, letting us run.

Since the `ids` array gets initialized to 0, the first push operation will not wait and immediately succeed, moving past the `while`. Then item is stored and finally 
the `ids[target]` is incremented. Since `id` is even `id + 1` is odd, allowing only next pop to succeed. This is like saying "I am done here, whicheever pop is after me, its your turn".

Here is an example concurrent call to first `pop` by thread1 followed by `push` by thread2 into an empty queue. We illustrate the execution of the two threads by these side by side blocks. The order of events progresses from top to bottom and is shared by the two threads, that is if statement1 of thread1 is above some other statement2 from thread2, then statement1 happened before statement2. If both statement1 and statement2 are on the same line then they happened without any particular ordering.

<table>
<tr>
<th>thread1 (pop)</th>
<th>thread2 (push)</th>
</tr>
<tr>
<td>
  
```C
pop(&q, &item) {
    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2;

    while(atomic_load(&q->ids[target]) != id); //waiting... 




    //finally!
    q->items[target] = item;
    atomic_store(&q->ids[target], id + 1);
}
```
</td>
<td>

```C
push(&q, 1) {

    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2 + 1;

    while(atomic_load(&q->ids[target]) != id); 

    *item_ptr = q->items[target];
    atomic_store(&q->ids[target], id + 2*QUEUE_CAP - 1);



}
```
</td>
</tr>
</table>

Similar execution will happen if thread1 pushes to a full queue, then thread2 pops first item. The pop will obtain `ticket = 0, target = 0, id = 1` succeed without waiting and finally set 
`ids[0] = id + 2*QUEUE_CAP - 1 == 2*QUEUE_CAP` which is precisely the id of push by thread1.  

In these posts I will be largely ignoring issues related to potential overflow. These issues are quite easy to solve and would just obfuscate the simplicity of the underlying ideas. Still however I will allow myself one optimalization which will drastically lower the probability of overflows of the each id in the ids array.

You might notice an inefficiency of the way we are assigning `id`: lets say a concurrent push and pop happened and one obtained ticket1 and the other ticket2. Unless ticket1 and ticket2 are some multiple of `QUEUE_CAP` appart from each othery, they will not share the same `target` and thus there is no reason for id1 to be disctinct from id2. Simply put there will never be a case where thread1 is waiting for thread2 to complete if their tickets are not multiple of `QUEUE_CAP` from each other. This means we can for pushes calculate id as `id = (ticket / QUEUE_CAP)*2` while for pops as `id = (ticket / QUEUE_CAP)*2 + 1`. This also allows us to change the increment after push/pop to just `atomic_store(&q->ids[target], id + 1)` for both.

> Of course in real implementation `QUEUE_CAP` will be a field of `Queue` which changes dynamically. Because of that, if you are like me, you might immediately get the urge to enforce `QUEUE_CAP` be power of two, thus allowing one to replace the dvision/remainder operations by `QUEUE_CAP` with bithshifts or mask operations. While this will of course improve the total running time of the critical section, the limitation to power of two capacity is perhaps too severe. Looking at Go these concurrent queues (channles) are often times *not* used to share data between threads but only to allow thread synchronization: thread1 pops and is blocked because queue is empty until some thread2 come along and pushes some dummy item. For this to work the the queue needs to be precisely some specific number. Using power of two greater than that will result in incorrect synchronization. 

## Closing

Okay so our queue is working, we have some producer happily producing items and some number of hungry consumers waiting to consume them. For concretness lets say the producer is reading a file: as soon as its done reading an item, it adds it into the queue from where some consumer takes it and parses it. Now the producer has successfully parsed the entire file and it exits. It would like to deinit the entire queue and move on about its life, but it cant: the consumers are still stuck waiting for more input! We need some mechanism to let all of the participating threads to stop whatever they are doing and exit. Just like Go we will call this operation close. To make our lives easier we will also add a constraint that after a channel is closed no operations can be perfromed with it.

Thankfully adding this closing mechanism is easy: just add a `bool closed` which is always checked in between spin waits and at the start of each function for good measure.

The resulting code (also incorporating the chnaged way to calculate `id`) looks like the following:

```C
#define QUEUE_CAP 10

typedef int Item; 

typedef struct Queue {
    _Atomic uint64_t head;
    _Atomic uint64_t tail;
    _Atomic bool closed;
    _Atomic uint32_t ids[QUEUE_CAP];
            Item items[QUEUE_CAP];
} Queue;

bool push(Queue* q, Item item) {
    if(atomic_load(&q->closed))
        return false;

    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2;

    while(atomic_load(&q->ids[target]) != id) {
        if(atomic_load(&q->closed)) 
            return false; 
    }

    q->items[target] = item;
    atomic_store(&q->ids[target], id + 1);
    return true;
}

bool pop(Queue* q, Item* item_ptr) {
    if(atomic_load(&q->closed))
        return false;

    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2 + 1;

    while(atomic_load(&q->ids[target]) != id) {
        if(atomic_load(&q->closed)) 
            return false; 
    }

    *item_ptr = q->items[target];
    atomic_store(&q->ids[target], id + 1);
    return true;
}

void close(Queue* q) {
    atomic_store(&q->closed, true);
}
bool is_closed(Queue* q) {
    return atomic_load(&q->closed);
}
```

Now all operations can also fail when interrupted by a concurrent call to `close`. The `atomic_fetch_sub` of head and tail simply the previously done increment. All good right?  

## Converged state

Up till now the queue appears simple: every access is guarded by a convenient ticket lock. Every operation goes nicely in order ensuring fairness, every operation is guranteed to finish leaving the queue in the correct state... wait! The last is no longer true! What does this mean?

First lets back off a bit and lets try to visualise a normal "converged" state of the queue. We will try to intuit some properties of this state. 
Look at the queue after performing the following block 
```C
Queue q = {0};
push(&q, 1);
push(&q, 2);
push(&q, 3);

Item into;
pop(&q, &into);
```

Now the queue in memory will look like the following (please excuse the ascii art):
```
U = uninitialized

        +---------+---------+---------+---------+
values: |    U    |    2    |    3    |   U..U  |
        +---------+---------+---------+---------+

        +---------+---------+---------+---------+
ids:    |    2    |    1    |    1    |   0..0  |
        +---------+---------+---------+---------+
                  ^                   ^         
                  head = 1            tail = 3
```

From this we can intuit the following rules:
0. head and tail are at max `QUEUE_CAP` appart from each other; tail is at least head
1. items in range `[head, tail)` contain user initialized data
2. for ticket in `[head, tail)`: ids are equal to `id = (ticket / QUEUE_CAP)*2 + 1` (filled)
3. for ticket in `[tail, head + QUEUE_CAP)`: ids are equal to `id = (ticket / QUEUE_CAP)*2` (not filled)

You can verify that this indeed does hold after all possible sequences of `push` and `pop` operations. This holds even for concurrent executions as long as we look at the queue after *all threads finish*. 

How does it look if some thread does not finish? I will asnwer that with the following example in which I will use different notation:
all thread execution will be written into the same code block but each line is prefixed with the thread which is executing said line. All threads still execute all their lines only their interleaving can be arbitrary. Again the vertical order determines the happens-before relation.

```C
t1: push(&q, 1) {
t1:     if(atomic_load(&q->closed))
t1:         return false;
t1: 
t1:     uint32_t ticket = atomic_fetch_add(&q->tail, 1);
t1:     uint32_t target = ticket % QUEUE_CAP;
t1:     uint32_t id = (ticket / QUEUE_CAP)*2;
t1:     //stopped midway through...     
t1:
t2: push(&q, 2); //entire function completes
```

The above code block is of course incomplete - thread t1 is nowhere near finishing its function, but alas lets look at hows the queue in memory looking.

```
U = uninitialized

        +---------+---------+---------+
values: |    U    |    2    |   U..U  |
        +---------+---------+---------+

        +---------+---------+---------+
ids:    |    0    |    1    |   0..0  |
        +---------+---------+---------+
        ^                   ^
        head = 0            tail = 2
```

The second push has succeeded and has already increment the id at `target = 1` while the first has not yet stored anything leaving its slot zero. This violates just about all of properties of converged state described above. 

Why am I saying this? It should be clear that if we just immedietely stop all execution at an arbtrary point in time with lets say... *ehm* the `close` operation, the resulting queue is not going to be in any sensible state. If allowed to use the queue after it was closed, we would surely run into problems (deadlocks, poping an item twice, overwriting present data). This is also the reason why the operation has to be `close` and not something like `cancel`, which would simply unblock all waiting threads without preventing any subsequent operations. The prevention of subsequent operation is not an arbitrary API design decision but an obligatery correctness feature.

Okay so we cannot use the queue after it was closed. Whats the big deal? We werent going to do that anyway, right? - Well we really wished to. Going back to the previous example of producer reading a file and consumers waiting to further process the contents, what if the relationship was inverted, the rate of production is greater then consumtption? The producer finishes reading the entire file and calls `close`, letting everyone exit... but also efectively destroying the contents of the queue! After `close` the queue can be in very invalid state, so reading from it can be problematic. 

So what to do about this? One might think that they can simply fix up the queue somehow, restoring it intos *some* functioning state. While that works, its rather costly O(n) operation and further the order of items does not have to be perserved, thus also breaking linearizibility. In next blog I will introduce a better model for closing, while also implementing proper futex based thread blocking.

> What is linearizibility? I wont give the fully general definition, since its not very useful. Instead for queues specifically it means that the following needs to hold: 
After thread1 calls `push(x)` followed by `push(y)`, any other thread2 which pops *both* of these values must also recieve them exactly in the correct order so `x == pop()` follwed by  `y == pop()`. The important bit is the both: there is no particular ordering if thread2 pops x and thread3 pops y. Perhpas more intuitivitely linearizibility just measn the queue perserves some notion of order, which is a property most people would expect.