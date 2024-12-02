# Fair, blocking concurrent queue (1)

Queues are the basic building block of concurrent comunication. One group of threads/processes/workers are producing data while other group is consuming data, forming a pipeline. This is pattern is happening in every layer of modern computing going down to even the hardware level, for example individual instructions are concurrently decoded, sheduled and executed with queues in between. 

The goal of this series of blogs will be to develop an easy to use concurrent queue primitive roughly equivalent to (buffered) Go channel. This type of queue has push/pop as its main operations, which add/remove item from the queue unless the queue is full/empty in which they block the calling thread untill it becomes not full/empty. Though unlike Go channels we wish to do this without locks. If the queue is not full it should be possible for two threads to push an item with minimal interference. 

Aside:
It may appear that having the block-on-full-push (implying there is finite capacity) limitation is too restrictive, dont we wish to grow for as long as we can? Well in practice no. Consider that if the production of new items is just a little faster then the consumption, the queue will eneventually fill up. Without a maximum capacity, it will keep on growing forever untill it exhausts significant portion of our memory, causing some potentially unrelated allocation to fail. This is problematic because it happens so easily - all it takes is a tiny bit slower consumer. A particular workload being slower to consumer? You already have a problem. Because all of this its easier to enforce some concrete upper bound. 

After some initial research I have found a quite excellent paper "T. R. W. Scogland - Design and Evaluation of Scalable Concurrent Queues for Many-Core Architectures, 2015" which can be found at https://synergy.cs.vt.edu/pubs/papers/scogland-queues-icpe15.pdf, which has all the properties we seek. We use this paper as the basis of our design and extend it in some important ways namely allowing full blocking instead of spin waiting, more useful closing behaviour. The remainder of this blog will be devoted to a hopefully intuitive explanation of the Scogland queue (queue from the paper).

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

void push(Queue *q, Item item) {
    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2;

    while(atomic_load(&q->ids[target]) != id);

    q->items[target] = item;
    atomic_store(&q->ids[target], id + 1);
}

void pop(Queue *q, Item* item_ptr) {
    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = ticket*2 + 1;

    while(atomic_load(&q->ids[target]) != id);

    *item_ptr = q->items[target];
    atomic_store(&q->ids[target], id + QUEUE_CAP + 1);
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

To start off this queue uses virtualized head/tail indices - this is to say that head/tail only ever increase and the actual index at which the item resides (`target`) is calculated by doing a modulo with the queue capacity. If `tail > head`, the queue stores precisley `tail - head` items. If `tail == head` the queue is empty. If this is novel to you I invite you to read about this approach here [link]. The reason why it was chosen is obvious - all we need to do to push item is a single atomic add instruction - no complicated ifs.

Okay those are the first two lines of the push and pop operations. What about the rest?

The queue holds an array `ids`, each of which act like a ticket lock. On a push operation the `tail` index is increment which yields `ticket` and as discussed above `target`. We also use `ticket` to calculate `id`. This id uniquelly represents this operation: 
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
atomic_store(&q->ids[target], id + QUEUE_CAP + 1);
```

</td>
</tr>
</table>



 


```C
#define QUEUE_CAP 10 //capacity
#define MAX_ID (UINT32_MAX/(QUEUE_CAP*2))

typedef int Item; 

typedef struct Queue {
    _Atomic uint32_t head;
    _Atomic uint32_t tail;
    _Atomic uint32_t ids[QUEUE_CAP];
            Item items[QUEUE_CAP];
} Queue;

void push(Queue *q, Item item) {
    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2;

    while(atomic_load(&q->ids[target]) != id);

    q->items[target] = item;

    uint32_t next_id = (id + 1) % MAX_ID;
    atomic_store(&q->ids[target], next_id);
}

bool pop(Queue *q, Item* item_ptr) {
    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2 + 1;

    while(atomic_load(&q->ids[target]) != id);

    *item_ptr = q->items[target];

    uint32_t next_id = (id + 1) % MAX_ID;
    atomic_store(&q->ids[target], next_id);
}

bool push(Queue *q, Item item) {
    if(atomic_load(&q->closed) != 0)
        return false;
    
    uint32_t ticket = atomic_fetch_add(&q->tail, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2;

    while(true) { 
        uint32_t curr_id = atomic_load(&q->ids[target]);
        if(curr == id)
            break;

        if(atomic_load(&q->closed) != 0) {
            atomic_fetch_sub(&q->tail, 1);
            return false; 
        }
        backoff(); 
    }

    q->items[target] = item;

    uint32_t next_id = (id + 1) % MAX_ID;
    atomic_store(&q->ids[target], next_id);
    return true;
}

bool pop(Queue *q, Item* item_ptr) {
    if(atomic_load(&q->closed) != 0)
        return false;
    
    uint32_t ticket = atomic_fetch_add(&q->head, 1);
    uint32_t target = ticket % QUEUE_CAP;
    uint32_t id = (ticket / QUEUE_CAP)*2 + 1;

    while(true) { 
        uint32_t curr_id = atomic_load(&q->ids[target]);
        if(curr == id)
            break;

        if(atomic_load(&q->closed) != 0) {
            atomic_fetch_sub(&q->head, 1);
            return false; 
        }
        backoff(); 
    }

    *item_ptr = q->items[target];

    uint32_t next_id = (id + 1) % MAX_ID;
    atomic_store(&q->ids[target], next_id);
    return true;
}


void backoff() {
    //x86-64 - tell the processort we are in a spin wait
    _mm_pause();
}

```