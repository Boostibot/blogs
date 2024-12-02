# Fair, blocking concurrent queue (1)

Queues are the basic building block of concurrent comunication. One group of threads/processes/workers are producing data while other group is consuming data, forming a pipeline. This is pattern is happening in every layer of modern computing going down to even the hardware level, for example individual instructions are concurrently decoded, sheduled and executed with queues in between. 

The goal of this series of blogs will be to develop an easy to use concurrent queue primitive roughly equivalent to (buffered) Go channel. This type of queue has push/pop as its main operations, which add/remove item from the queue unless the queue is full/empty in which they block the calling thread untill it becomes not full/empty. Though unlike Go channels we wish to do this without locks. If the queue is not full it should be possible for two threads to push an item with minimal interference. 

After some initial research I have found a quite excellent paper [T. R. W. Scogland - Design and Evaluation of Scalable Concurrent Queues for Many-Core Architectures, 2015](https://synergy.cs.vt.edu/pubs/papers/scogland-queues-icpe15.pdf), which has all the properties we seek. We use this paper as the basis of our design and extend it in some important ways namely allowing full blocking instead of spin waiting, more useful closing behaviour. The remainder of this blog will be devoted to a hopefully intuitive explanation of the Scogland queue (queue from the paper).

> It may appear that having the block-on-full-push (implying there is finite capacity) limitation is too restrictive, dont we wish to grow for as long as we can? Well in practice no. Consider that if the production of new items is just a little faster then the consumption, the queue will eneventually fill up. Without a maximum capacity, it will keep on growing forever untill it exhausts significant portion of our memory, causing some potentially unrelated allocation to fail. This is problematic because it happens so easily - all it takes is a tiny bit slower consumer. A particular workload being slower to consumer? You already have a problem. Because all of this its easier to enforce some concrete upper bound. 

## First try

I assume the reader is more familiar with actual code then some abstract pseudocode so I will just give the full code from the paper. The code is slightly altered to be compilable, use C11/C++ atomics, do just one operation per line (thus be easier to describe). I have also fixed one misstake present in the code listing. 

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

Except few key details, this is the data structure presented in the paper. We will expand it to further down the post. 

So what is this doing? 

To start off this queue uses virtualized head/tail indices - this is to say that head/tail only ever increase and the actual index at which the item resides (`target`) is calculated by doing a modulo with the queue capacity. If `tail > head`, the queue stores precisley `tail - head` items. If `tail == head` the queue is empty. If this is novel to you I invite you to read about this approach [here](https://fgiesen.wordpress.com/2010/12/14/ring-buffers-and-queues/). The reason why it was chosen is obvious - all we need to do to push item is a single atomic add instruction - no complicated ifs.

Okay those are the first two lines of the push and pop operations. What about the rest?

The queue holds an array `ids`, each of which act like a [ticket lock](https://mfukar.github.io/2017/09/08/ticketspinlock.html). On a push operation the `tail` index is increment which yields `ticket` and as discussed above `target`. We also use `ticket` to calculate `id`. This id uniquelly represents this operation: 
- the id is always (atomically) increasing so no two pushesh can have the same one (ignoring overflow) 
- ids of push operations are even (`*2`) while pop operations are odd (`*2 + 1`)

Now that we have obtained our unique `id` and some `target` we wait for the `ids[target]` to become our `id`. We for now do a simple spin loop whic is like repeatedly asking "Is it my turn? Is it my turn?", while hoepfully some other thread eventually responds, letting us run.

Since the `ids` array gets initialized to, the first push operation will not wait and immediately succeed, moving past the while. Then item is stored and finally 
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
uint32_t ticket = atomic_fetch_add(&q->tail, 1);
uint32_t target = ticket % QUEUE_CAP;
uint32_t id = ticket*2;

while(atomic_load(&q->ids[target]) != id); //waiting... 




//finally!
q->items[target] = item;
atomic_store(&q->ids[target], id + 1);
```
  
</td>
<td>

```C
uint32_t ticket = atomic_fetch_add(&q->head, 1);
uint32_t target = ticket % QUEUE_CAP;
uint32_t id = ticket*2 + 1;

while(atomic_load(&q->ids[target]) != id); 

*item_ptr = q->items[target];
atomic_store(&q->ids[target], id + 2*QUEUE_CAP - 1);
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

Now all operations can also fail when interrupted by a concurrent call to `close`. All good right?  

## A critical look

Up till now the queue appears simple: every access is guarded by a convenient ticket lock. Every operation goes nicely in order ensuring fairness, every operation is guranteed to finish leaving the queue in the correct state... wait! The last is no longer true! What does this mean?

Lets answer that first with an illustration of potential concurrent execution resulting in some problematic cases. Because there are more threads then previous I will use a different format:
all thread execution will be written into the same code block but each line is prefixed with the thread which is executing said line. All threads still execute all their lines only their interleaving can be arbitrary. Again the vertical order determines the happens-before relation.

```C

```

> This is also the answer to the question of why the operation has to be `close` and not something like `cancel`, which would simply unblock all waiting threads without preventing any subsequent operations. The prevention of subsequent operation is not an arbitrary API design decision but an obligatery correctness feature - without it we may pop items that were never inserted, pop a given item twice or similar!

## Converged state
