# COMP3891 Extended Operating Systems 2017/S1 - Assignment 1 (Synchronisation)

> In this assignment you will solve a number of synchronisation and locking problems. You will also get experience with data structure and resource management issues.

# Part 1 

This issue in this part is that the counter variable and
the adder_counters array are shared resources among threads, and
it is being accessed by all running threads when they need it. This
results in concurrency issues in regards to correct counting, when
the resource is accessed by multiple threads at once.

The adder() function contains the critical section of code that must
lock resources when being used.

The critical region is deemed to be within a single iteration of the
while loop (line 78 in our solution). The resources which are shared
are locked with lockA, and released at the loop iteration end.

# Part 2 

This issue with with part is that a deadlock is created when
thread bill() locks lock a, and then b. This is an issue because at
the same time; thread ben() locks b and then a.

This can happen at the same time, so that bill has lock a, and is
blocked on requesting lock b, and ben has lock b and is blocked
in requesting lock a. This is a circular blocking request and the
program deadlocks.

To prevent this, threads must lock resources in some pre-decided order
which is consistent throughout the program. So all threads must lock
'lock a' first, and the lock b - releasing of the locks should still
be done in nested order (EG request lock a, then b. Release lock b,
then a).  This will prevent the deadlock from occurring.

The critical region in each thread is the unused pointers being
casted, which does nothing. So really there were not any global
shared resources being accessed within the critical regions. These
critical regions in each function bill() and ben() lie within a for
loop however. So resources are requested and released on every loop
iteration.

The fix here is to make sure all resources are requested in the
same order, so ben() must request lock a, then b. And not the other
way around.

# Part 3

This task involved writing functions, rather than editing existing
ones with concurrency issues / deadlock problems.

The shared resources across threads in this Part involve a data item
buffer (buffer[BUFFER_SIZE]), and two indicies with index the start
and end of this buffer (bufStart, bufEnd).

I will split this discussion into functions:

## consumer_receive()

When the function starts, the semaphore consumer_hold is attempted
to be secured (decremented), and if the value is already 0, then
the thread is blocked. This semaphore has a starting value of 0, and
represents the number of occupied places in the buffer array. Therefore
when a producer calls V(consumer_hold) when it has added an item,
a blocked consumer may then receive and process the item.

The critical region of this function is the following: 
32 thedata = buffer[bufStart]; 
33 bufStart = (bufStart+1) % BUFFER_SIZE;

The buffer is being accessed using the start index, and then then
this index is being updated. This critical region is locked using
the bufLock lock, as to prevent concurrency issues between threads.

After this processing, the consumer then signals V(producer_hold),
incrementing the producer_hold semaphore which represents the number
of free spots in the buffer array. This means a block producer may
add an item to the array.

## producer_send()

This function is very similar to the above.

When the function starts, the semaphore producer_hold is attempted
to be secured (decremented), and if the value is already 0, then the
thread is blocked. This semaphore has a starting value of BUFFER_SIZE,
and represents the number of free places in the buffer array. Therefore
when a receiver calls V(producer_hold) when it has taken an item,
a blocked producer may then add another item to the buffer array.

The critical region of this function is the following: 
51 bufEnd = (bufEnd + 1) % BUFFER_SIZE; 
52 buffer[bufEnd] = item;

The end index is being updated, and then the buffer is being accessed
using the end index. This critical region is locked using the bufLock
lock, as to prevent concurrency issues between threads.

After this processing, the producer then signals V(consumer_hold),
incrementing the consumer_hold semaphore which represents the number
of occupied spots in the buffer array. This means a block consumer
may take an item from the array.

# Part 4

This part is very similar to Part 3 in that we have producers and
consumers (customers and staff).

Our shared resources in here are orderBuf (an array of paintorder
structs), and again two indicies to index this buffer (bufStart
and bufEnd).

## order_paint()

Almost the exact same as part 3, we decrement a semaphore buffer_hold
which represents the number of free spaces in the orderBuf order
buffer. If the semaphore is already 0, then the customer is blocked
until a staff member finishes an order and calls V(buffer_hold),
symbolizing that a space on the orderBuf array has been cleared
up. This semaphore has a starting value of NCUSTOMERS, as there are
NCUSTOMER free spaces in the order buffer to start.

The critical region of this function is the following: 
55 bufEnd = (bufEnd + 1) % NCUSTOMERS; 
56 order->order_owner = bufEnd;
57 orderBuf[bufEnd] = order;

The end index is being updated, and then the order that is being
constructed is assigned an order_owner ID, which is the position in
the orderBuf array, and then the buffer is being accessed using the
end index to insert an order. This critical region is locked using
the bufLock lock, as to prevent concurrency issues between threads.

After this processing, the customer then signals V(order_hold),
incrementing the order_hold semaphore which represents the number of
occupied spots in the buffer array. This means a blocked staff may
take an order from the array.

Finally the customer then blocks his/her self, using the order_owner
id (the index of the customer's order in the orderBuf). The customer
will return only when the staff member that services the customers
order signals at the end of the order's processing.

## paintorder()

We decrement a semaphore order_hold which represents the number of
occupied spaces in the orderBuf order buffer. If the semaphore is
already 0, then the staff member is blocked until a customer places
an order and calls V(order_hold), symbolizing that an order has
been placed in orderBuf. This semaphore has a starting value of 0,
as there are 0 occupied spaces in the order buffer to start.

The critical region of this function is the following: 
90 struct paintorder *ret = orderBuf[bufStart]; 
91 bufStart = (bufStart + 1) % NCUSTOMERS;

An order is pulled from the orderBuf array using the bufStart index,
and then this index is incremented. This critical region is locked
using the bufLock lock, as to prevent concurrency issues between
threads.

After this processing, the staff member then signals V(buffer_hold),
incrementing the buffer_hold semaphore which represents the number
of free spots in the buffer array. This means a blocked customer may
place an order in the array.

## fill_order()

This function pulls the requested tints from the passed-in order
and if the tint is an actual tint, then the tint_hold lock array is
indexed for the respective tint, and the lock acquired. This is to
prevent blocking as best as can be for the mix(order) function.

On finishing the mix(order) function, all the tint locks that were
held are released.

## serve_order()

In this function, the order that has been completed is used to find
the order_owner ID, so that the customer that is blocked on this
semaphore can be signaled to return from order_paint() 

