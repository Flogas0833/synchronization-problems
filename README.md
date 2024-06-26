# Synchronization problems

Problems commonly found when implementing concurrency and parallelism.

Students:
- Nguyen Thai Hoa 20224850
- Nguyen Thai Khoi 20224868
- Vu Tung Lam 20225140
- Dao Phuc Long 20220034

## Concurrency
In computer science, concurrency is the ability of different parts or units of a program, algorithm, or problem to be executed out-of-order or in partial order, without affecting the outcome. This allows for parallel execution of the concurrent units, which can significantly improve overall speed of the execution in multi-processor and multi-core systems. In more technical terms, concurrency refers to the decomposability of a program, algorithm, or problem into order-independent or partially-ordered components or units of computation.

According to Rob Pike, concurrency is the composition of independently executing computations, and concurrency is not parallelism: concurrency is about dealing with lots of things at once but parallelism is about doing lots of things at once. Concurrency is about structure, parallelism is about execution, concurrency provides a way to structure a solution to solve a problem that may (but not necessarily) be parallelizable.

The following table compares the differences between different forms of execution in a multi-core machine.

|             | Singlethreading (synchronous) | Singlethreading (asynchronous) | Multithreading | Multiprocessing |
| ----------- | :---------------------------: | :----------------------------: | :------------: | :-------------: |
| Concurrency | | x | x | x |
| Parallelism | | | x | x |

## Synchronization primitives

### Event

Event object is used to notify multiple threads that an event has happened. It usually has 3 public methods: `wait`, `set`, `clear`.

An event object manages an internal flag. If the flag is `true`, any calls to `wait` will return immediately. If the flag is `false`, the wait method will suspend the current thread and wait for the flag to become `true` before returning.

The internal flag can be switched by `set` and `clear` methods.

### Lock

Mutex lock to guarantee exclusive access to a shared state. It usually has 2 public methods: `acquire` and `release`. A Lock object can be in one of two states: "locked" or "unlocked".

If the lock is "locked", all threads that call `acquire` will be put in a waiting FIFO queue and will proceed in order for each release call.

If the lock is "unlocked", calling `acquire` will set the lock to the "locked" state and return immediately.

The `release` method will release the lock held by the current thread. If the lock isn't acquired then this method does nothing.

### Semaphore

A semaphore object that allows a limited number of threads to acquire it. Similar to a lock, the semaphore usually has 2 public methods: `acquire` and `release`.

A semaphore is a synchronization primitive that maintains a counter indicating the number of available resources or permits. In this implementation, the semaphore keeps track of an internal counter. The counter is decremented each time a future acquires the semaphore using the `acquire` method and incremented each time the semaphore is released using the `release` method.

When multiple threads are waiting for the semaphore, they will be put in a FIFO queue and only the first one will proceed when the semaphore becomes available.

If the internal counter of the semaphore has an upper limit of 1, it is identical to a lock.

### Condition

A condition variable is always associated with some kind of lock; this can be passed in or one will be created by default. Passing one in is useful when several condition variables must share the same lock. The lock is part of the condition object: we do not have to track it separately.

Other methods must be called with the associated lock held. The `wait()` method releases the lock, and then blocks until another thread awakens it by calling `notify()` or `notify_all()`. Once awakened, `wait()` re-acquires the lock and returns. It is also possible to specify a timeout.

The `notify()` method wakes up one of the threads waiting for the condition variable, if any are waiting. The `notify_all()` method wakes up all threads waiting for the condition variable.

Note: the `notify()` and `notify_all()` methods do not release the lock; this means that the thread or threads awakened will not return from their `wait()` call immediately, but only when the thread that called `notify()` or `notify_all()` finally relinquishes ownership of the lock.

The typical programming style using condition variables uses the lock to synchronize access to some shared state; threads that are interested in a particular change of state call `wait()` repeatedly until they see the desired state, while threads that modify the state call `notify()` or `notify_all()` when they change the state in such a way that it could possibly be a desired state for one of the waiters. For example, the following code is a generic producer-consumer situation with unlimited buffer capacity:

```cpp
Condition condition = Condition(Lock());

// Consume one item
condition.acquire();
while (!an_item_is_available())
{
    condition.wait();
}
get_an_available_item();
condition.release();

// Produce one item
condition.acquire();
make_an_item_available();
condition.notify();
condition.release();
```

## Implementation

All testing are done using C++17 (compiled with the `-O0` flag). We utilize the semaphore and event objects from the win32 C++ API and wrap in custom classes like below:

Wrapper class for `Event` objects:
```cpp
class Event
{
private:
    HANDLE _event;

public:
    Event(bool flag) : _event(CreateEventW(NULL, TRUE, flag, NULL)) {}
    ~Event()
    {
        CloseHandle(_event);
    }

    void set() const
    {
        SetEvent(_event);
    }

    void clear() const
    {
        ResetEvent(_event);
    }

    void wait() const
    {
        WaitForSingleObject(_event, INFINITE);
    }
};
```

Wrapper class for `Semaphore` and `Lock` objects:
```cpp
class Semaphore
{
private:
    HANDLE _semaphore;

public:
    Semaphore(int count) : _semaphore(CreateSemaphoreW(NULL, count, count, NULL)) {}
    ~Semaphore()
    {
        CloseHandle(_semaphore);
    }

    void acquire() const
    {
        WaitForSingleObject(_semaphore, INFINITE);
    }

    void release(LONG count = 1) const
    {
        ReleaseSemaphore(_semaphore, count, NULL);
    }
};

class Lock : public Semaphore
{
public:
    Lock() : Semaphore(1) {}

    void release() const
    {
        Semaphore::release();
    }
};
```

Additionally, we implement a `Condition` class as below:
```cpp
class Condition
{
private:
    const Lock *_lock;
    std::deque<Lock> _waiters;

public:
    Condition(const Lock *lock) : _lock(lock) {}

    void acquire()
    {
        _lock->acquire();
    }

    void release()
    {
        _lock->release();
    }

    void wait()
    {
        Lock waiter = Lock();
        waiter.acquire();
        _waiters.push_back(waiter);

        release();
        waiter.acquire();
    }

    void notify(int n)
    {
        for (int i = 0; i < n; i++)
        {
            if (_waiters.empty())
            {
                return;
            }

            auto front = _waiters.front();
            _waiters.pop_front();
            front.release();
        }
    }

    void notify_all()
    {
        notify(_waiters.size());
    }
};
```

## ABA Problem

The ABA problem is a subtle challenge that can arise in multithreaded programming when dealing with shared memory. It occurs during synchronization, specifically when relying solely on a variable's current value to determine if data has been modified.

The ABA problem occurs when multiple threads (or processes) accessing shared data interleave. Below is a sequence of events that illustrates the ABA problem:

1. Process $P_1$ reads value A from some shared memory location.

2. $P_1$ is preempted, allowing process $P_2$ to run.

3. $P_2$ writes value B to the shared memory location.

4. $P_2$ writes value A to the shared memory location.

5. $P_2$ is preempted, allowing process $P_1$ to run.

6. $P_1$ reads value A from the shared memory location.

7. $P_1$ determines that the shared memory value has not changed and continues.

### Example

An example illustrating how the ABA problem occurs:

```cpp
class __StackNode
{
public:
    const int value;
    bool valid = true;  // for demonstration purpose
    __StackNode *next;

    __StackNode(int value, __StackNode *next) : value(value), next(next) {}
    __StackNode(int value) : __StackNode(value, nullptr) {}
};

class Stack
{
private:
    std::atomic<__StackNode *> _head_ptr = nullptr;

public:
    /** Pop the head object and return its value */
    int pop(bool sleep)
    {
        while (true)
        {
            __StackNode *head_ptr = _head_ptr;
            if (head_ptr == nullptr)
            {
                throw std::out_of_range("Empty stack");
            }

            // For simplicity, suppose that we can ensure that this dereference is safe
            // (i.e., that no other thread has popped the stack in the meantime).
            __StackNode *next_ptr = head_ptr->next;

            if (sleep)
            {
                Sleep(200); // interleave
            }

            // If the head node is still head_ptr, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head node with next.
            if (_head_ptr.compare_exchange_weak(head_ptr, next_ptr))
            {
                int result = head_ptr->value;
                head_ptr->valid = false; // mark this memory as released
                delete head_ptr;
                return result;
            }
            // The stack has changed, start over.
        }
    }

    /** Push a value to the stack */
    void push(int value)
    {
        __StackNode *value_ptr = new __StackNode(value);
        while (true)
        {
            __StackNode *next_ptr = _head_ptr;
            value_ptr->next = next_ptr;
            // If the head node is still next, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head with value.
            if (_head_ptr.compare_exchange_weak(next_ptr, value_ptr))
            {
                return;
            }
            // The stack has changed, start over.
        }
    }

    void print()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            std::cout << current->value << " ";
            current = current->next;
        }
        std::cout << std::endl;
    }

    void assert_valid()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            if (!current->valid)
            {
                throw std::runtime_error("Invalid stack node");
            }
            current = current->next;
        }
    }
};

DWORD WINAPI thread_proc(void *_stack)
{
    Stack *stack = (Stack *)_stack;
    stack->pop(false);
    stack->pop(false);
    stack->push(40);

    return 0;
}

int main()
{
    Stack stack;
    stack.push(10);
    stack.push(20);
    stack.push(30);
    stack.push(40);

    HANDLE thread = CreateThread(
        NULL,         // lpThreadAttributes
        0,            // dwStackSize
        &thread_proc, // lpStartAddress
        &stack,       // lpParameter
        0,            // dwCreationFlags
        NULL          // lpThreadId
    );

    stack.pop(true);

    WaitForSingleObject(thread, INFINITE);
    CloseHandle(thread);

    stack.print();
    stack.assert_valid();  // randomly crash

    return 0;
}
```

In the example above, 2 threads are allowed to run concurrently. To promote the occurrence of the ABA problem, the main thread calls `stack.pop(true)` with `Sleep(200)` to suspend its execution, allowing other threads to run. Before interleaving the main thread, the local pointer variable `head_ptr` and `next_ptr` point to `40` and `30`, respectively. We can assume that while the main thread is sleeping, the second one completes all of its operations (2 `pop`s and 1 `push`), transforms the stack into `[10, 20, 40]` and deallocates the memory of the node `30`. Afterwards, the main thread resumes its execution, where the method call `_head_ptr.compare_exchange_weak(head_ptr, next_ptr)` returns `true` (since the top of the stack is still `40`). It then sets `next_ptr` (which is a pointer to `30`) as the top of the stack. But since this memory is deallocated, it is unsafe to perform the assignment. Thus running the program above will randomly crash at the line `stack.assert_valid()`.

It is worth noticing that since the memory pointed by `next_ptr` may not be touched anyway after deallocation, assigning `next_ptr` in the main thread still appears to work correctly (in this example, we have to clear a flag `valid` to mark the node as deallocated). Accessing freed memory is undefined behavior: this may result in crashes, data corruption or even just silently appear to work correctly.

#### Workarounds

A common workaround is to add extra "tag" or "stamp" bits to the quantity being considered. For example, an algorithm using compare and swap on a pointer might use the low bits of the address to indicate how many times the pointer has been successfully modified. Because of this, the next compare-and-swap will fail, even if the addresses are the same, because the tag bits will not match. This is sometimes called ABAʹ since the second A is made slightly different from the first. Such tagged state references are also used in transactional memory. Although a tagged pointer can be used for implementation, a separate tag field is preferred if double-width CAS is available.

In the example above, it is possible to add a `version` field to the `Stack`, indicating the number of times the stack has been updated. Instead of comparing `_head_ptr.compare_exchange_weak(next_ptr, value_ptr)`, we can compare the stack's version before and after performing assignments.

### Addressing the ABA Problem

Several approaches can help mitigate the ABA problem:

* **Timestamps:** By associating a version number or timestamp with the data, threads can ensure the value hasn't changed between reads. An inconsistency arises if the version retrieved during the second read doesn't match the version associated with the initial value.
* **Locking mechanisms:** Utilizing synchronization primitives like mutexes or spinlocks ensures exclusive access to the shared memory location during critical sections. This prevents other threads from modifying the data while the first thread is performing its read-modify-write operation.
* **Atomic operations:** In specific scenarios, employing atomic operations that combine read and write into a single, indivisible step can eliminate the window of vulnerability between reads. However, atomic operations might not be suitable for all situations.

By understanding the ABA problem and implementing appropriate synchronization techniques, developers can ensure the integrity of data in multithreaded environments.

## Sleeping Barber Problem

The Sleeping Barber problem is a classic synchronization problem that illustrates the challenges of process management, specifically around CPU scheduling, resource allocation, and deadlock prevention.
The scenario is as follows:
* A barber works in a barbershop with one barber chair and a waiting room with several chairs.
* If there are no customers, the barber goes to sleep.
* When a customer arrives, if the barber is asleep, the customer wakes up the barber.
* If the barber is busy, the customer sits in the waiting room.
* If the waiting room is full, the customer leaves.

There are two main complications. First, there is a risk that a race condition, where the barber sleeps while a customer waits for the barber to get them for a haircut, arises because all of the actions - checking the waiting room, entering the shop, taking a waiting room chair - take a certain amount of time. Specifically, a customer may arrive to find the barber cutting hair so they return to the waiting room to take a seat but while walking back to the waiting room the barber finishes the haircut and goes to the waiting room, which he finds empty (because the customer walks slowly) and thus goes to sleep in the barber chair. Second, another problem may occur when two customers arrive at the same time when there is only one empty seat in the waiting room and both try to sit in the single chair; only the first person to get to the chair will be able to sit.

A multiple sleeping barbers problem has the additional complexity of coordinating several barbers among the waiting customers.
### Example

```cpp
#include <iostream>
#include <vector>
#include <windows.h>

HANDLE barber_ready = CreateSemaphoreW(NULL, 0, 1, NULL);
HANDLE read_write_seats = CreateSemaphoreW(NULL, 1, 1, NULL);
HANDLE ready_customers = CreateSemaphoreW(NULL, 0, 1, NULL);

int free_seats = 3; // assume there are 3 seats in the waiting room

DWORD WINAPI barber(void *)
{
    while (true)
    {
        WaitForSingleObject(ready_customers, INFINITE);
        WaitForSingleObject(read_write_seats, INFINITE);

        free_seats++;

        ReleaseSemaphore(barber_ready, 1, NULL);
        ReleaseSemaphore(read_write_seats, 1, NULL);

        // cut hair
        std::cout << "Cutting hair\n";
    }
}

DWORD WINAPI customer(void *)
{
    while (true)
    {
        WaitForSingleObject(read_write_seats, INFINITE);
        if (free_seats > 0)
        {
            free_seats--;
            ReleaseSemaphore(ready_customers, 1, NULL);
            ReleaseSemaphore(read_write_seats, 1, NULL);
            WaitForSingleObject(barber_ready, INFINITE);

            // have a hair cut
        }
        else
        {
            ReleaseSemaphore(read_write_seats, 1, NULL);
            std::cout << "No empty slots, leaving...\n";
        }
    }
}

int main()
{
    HANDLE barber_thread = CreateThread(NULL, 0, &barber, NULL, 0, NULL);

    std::vector<HANDLE> customer_threads;
    for (int i = 0; i < 4; i++)
    {
        customer_threads.push_back(CreateThread(NULL, 0, &customer, NULL, 0, NULL));
    }

    WaitForSingleObject(barber_thread, INFINITE);
    for (auto &thread : customer_threads)
    {
        WaitForSingleObject(thread, INFINITE);
    }

    return 0;
}
```

In the example above:
* The barber thread waits for customers and cuts their hair.
* Customers arrive, check if there’s space in the waiting room, and either wait or leave.
* Semaphores (`cv_barber` and `cv_customer`) are used for synchronization.

### Real-world implications

* **Customer Service Queues:** Similar to the barbershop scenario, customer service centers often have a limited number of representatives to handle a fluctuating number of customer calls or requests. The challenge is to efficiently manage the queue so that customers are served in a timely manner without overwhelming the service reps.
* **Computer Operating Systems:** In operating systems, managing multiple processes that need access to limited CPU time or memory resources is akin to the Sleeping Barber problem. The OS must ensure that each process gets a fair chance to execute without causing deadlock or starvation.
* **Database Access:** Multiple applications or users trying to access and modify a database can lead to concurrency issues. The Sleeping Barber problem helps in designing systems that prevent conflicts and ensure data integrity when multiple transactions occur simultaneously.
* **Airport Runway Scheduling:** Air traffic controllers must manage the takeoffs and landings of aircraft on a limited number of runways. The principles derived from the Sleeping Barber problem can help in creating schedules that maximize runway usage while maintaining safety.
* **Hospital Emergency Rooms:** In healthcare, particularly in emergency rooms, patients must be attended based on the severity of their condition. The Sleeping Barber problem’s solutions can inform the design of triage systems that manage patient flow effectively.
* **Web Server Management:** Web servers handling requests for web pages or services must manage their threads to respond to simultaneous requests. The Sleeping Barber problem provides insights into how to balance load without causing long wait times or server crashes.

### Addressing the Sleeping Barber Problem

Several approaches can help mitigate the Sleeping Barber problem:
* **Semaphores:** The most common solution involves using semaphores to coordinate access to resources. Semaphores can be used to signal the availability of the barber (or resource) and the waiting chairs (queue space). This ensures that customers (or tasks) are served in the order they arrive and that the barber (or resource) is not overwhelmed.
* **Mutexes:** Mutexes are used to ensure mutual exclusion, particularly in critical sections of the code where variables are accessed by multiple threads. This prevents race conditions and ensures that the system’s state remains consistent.
* **Condition Variables:** These are used in conjunction with mutexes to block a thread until a particular condition is met. For example, a barber thread might wait on a condition variable until there is a customer in the waiting room.
* **Monitor Constructs:** Monitors are higher-level synchronization constructs that provide a mechanism for threads to temporarily give up exclusive access in order to wait for some condition to be met, without risking deadlock or busy-waiting.
* **Message Passing:** In distributed systems, message passing can be used to synchronize processes. This involves sending messages between processes to signal the availability of resources or the need for service.
* **Event Counters:** These can be used to keep track of the number of waiting customers and to signal the barber when a customer arrives. This helps in managing the queue and ensuring that no customer is missed.
* **Ticket Algorithms:** Ticket algorithms can be used to assign a unique number to each customer, ensuring that service is provided in the correct order. This is similar to taking a number at a deli counter.

By understanding the Sleeping Barber problem and implementing appropriate synchronization techniques, developers  learn how to efficiently manage limited resources in concurrent systems. 

## Cigarette smokers problem

The cigarette smokers problem is a classic concurrency problem in computer science. This problem highlights the challenges of coordinating multiple processes or threads that share resources and need to synchronize their actions.

Here is the description for this problem
* **Ingredients:** Imagine a scenario where there are three ingredients required to make and smoke a cigarette: tobacco, paper, and matches.
* **Participants:** Around a table, there are three smokers, each of whom has an infinite supply of one of the three ingredients:
  1. One smoker has an infinite supply of tobacco.
  2. Another smoker has an infinite supply of paper.
  3. The third smoker has an infinite supply of matches.
* **Non-smoking agent:** There is also a non-smoking agent who enables the smokers to make their cigarettes. The agent randomly selects two of the supplies and places them on the table.
* **Smoking process:**
  1. The smoker who has the third supply should remove the two items from the table.
  2. They use these items (along with their own supply) to make a cigarette, which they smoke for a while.
  3. Once the smoker finishes smoking, the agent places two new random items on the table.
  4. This process continues indefinitely.

### Example

A naive implementation given below, where each smoker waits for the other 2, will easily suffer from deadlock.

```cpp
std::mt19937 rng(std::chrono::steady_clock::now().time_since_epoch().count());

template <typename T>
T random_int(const T &l, const T &r)
{
    std::uniform_int_distribution<T> unif(l, r);
    return unif(rng);
}

const Lock smoking = Lock();
std::vector<Lock> locks(3);

void initialize()
{
    smoking.acquire();
    for (int i = 0; i < 3; i++)
    {
        locks[i].acquire();
    }
}

DWORD WINAPI agent(void *)
{
    while (true)
    {
        int ingredient = random_int(0, 2), next_ingredient = (1 + ingredient) % 3;
        std::cout << "Got ingredients " << ingredient << ", " << next_ingredient << std::endl;

        locks[ingredient].release();
        locks[next_ingredient].release();
        smoking.acquire();
    }

    return 0;
}

DWORD WINAPI smoker(void *ptr)
{
    int ingredient = *(int *)ptr;
    while (true)
    {
        locks[(ingredient + 1) % 3].acquire();
        std::cout << "Smoker " << ingredient << " got " << (ingredient + 1) % 3 << std::endl;

        locks[(ingredient + 2) % 3].acquire();
        std::cout << "Smoker " << ingredient << " got " << (ingredient + 2) % 3 << std::endl;

        std::cout << "Smoker " << ingredient << " is smoking" << std::endl;
        Sleep(500);
        std::cout << "Smoker " << ingredient << " is done" << std::endl;

        smoking.release();
    }

    return 0;
}

int main()
{
    initialize();

    std::vector<HANDLE> threads;
    threads.push_back(
        CreateThread(
            NULL,   // lpThreadAttributes
            0,      // dwStackSize
            &agent, // lpStartAddress
            NULL,   // lpParameter
            0,      // dwCreationFlags
            NULL)   // lpThreadId
    );

    int *ingredient_ptr[3];
    for (int i = 0; i < 3; i++)
    {
        ingredient_ptr[i] = new int(i);
        threads.push_back(CreateThread(NULL, 0, &smoker, ingredient_ptr[i], 0, NULL));
    }

    for (auto &thread : threads)
    {
        WaitForSingleObject(thread, INFINITE);
        CloseHandle(thread);
    }

    for (int i = 0; i < 3; i++)
    {
        delete ingredient_ptr[i];
    }

    return 0;
}
```

### Real-world implications

The Cigarette Smokers Problem in computer science, beyond being a theoretical exercise, has real-world implications, particularly in the field of concurrent programming and operating systems. Here are some of the key implications:
* **Deadlock prevention** The problem demonstrates a scenario where deadlock can occur if resources are not managed properly. In real-world systems, such as databases and operating systems, managing access to shared resources is crucial to prevent deadlock, which can cause systems to halt or become unresponsive.
* **Resource allocation** It illustrates the challenges in allocating limited resources among competing processes or threads. This is analogous to real-world situations where multiple applications or users require access to a finite set of resources, such as CPU time, memory, or network bandwidth.
* **Synchronization mechanisms** The problem highlights the importance of proper synchronization mechanisms, like semaphores, locks, and condition variables, to coordinate the actions of concurrent processes. These mechanisms are widely used in developing multi-threaded applications, ensuring that processes operate in the correct sequence without interfering with each other.
* **System design** It emphasizes the need for careful system design to avoid complex interdependencies that can lead to deadlock. This is relevant for designing systems that are robust, scalable, and maintainable.
* **Understanding concurrency** The problem serves as an educational tool to help programmers understand the complexities of concurrency, which is essential for developing efficient and reliable software in a multi-core and distributed computing world.
* **Semaphore limitations** The problem also points out the limitations of traditional semaphores and the need for more powerful synchronization primitives in certain scenarios.

### Addressing the Cigarette Smokers Problem

Several approaches can help mitigate the Cigarette Smokers Problem:
* **Deadlock avoidance:** Implement a deadlock avoidance strategy to prevent the system from entering a deadlock state.
* **Priority-based allocation:** Assign priorities to processes or threads. When allocating resources, give preference to higher-priority processes. This helps prevent low-priority processes from blocking critical resources indefinitely.
* **Resource pooling:** Create a pool of resources (e.g., semaphores or locks) that processes can request. When a process is done, it releases the resource back to the pool. This avoids resource exhaustion and ensures fair access.
* **Two-phase locking:** In database management systems, use two-phase locking to ensure that transactions acquire and release locks in a consistent order. This helps prevent deadlocks during data updates.
* **Timeouts and rollbacks:** If a process waits too long for a resource, introduce a timeout. If the timeout expires, the process releases its resources and rolls back its work. This prevents indefinite waiting.
* **Resource hierarchies:** Assign a hierarchy to resources (e.g., locks). Processes must acquire resources in a specific order (from lower to higher levels). This prevents circular waits.
* **Dynamic resource allocation:** Dynamically allocate resources based on demand. For example, allocate memory or threads as needed and release them when no longer required.
* **Avoidance of hold-and-wait:** Processes should request all required resources upfront (non-preemptive). If a process cannot acquire all resources, it releases any acquired resources and retries later.
* **Preemption:** If a high-priority process needs a resource held by a lower-priority process, preempt the lower-priority process and allocate the resource to the higher-priority one.
* **Transaction serialization:** In database systems, ensure that transactions are serialized (executed one after the other) to avoid conflicts and deadlocks.

In summary, the Cigarette Smokers Problem is not just a theoretical construct but a representation of the real challenges faced in concurrent programming and system design. It encourages developers to think critically about process synchronization, resource allocation, and system robustness in the context of concurrent operations.

### Barrier Synchronization Problem

Barrier synchronization problems arise in parallel computing when multiple threads or processes must reach a synchronization point (or barrier) at the same time. The primary purpose of a barrier is to ensure that no thread or process proceeds beyond a certain point until all others have reached that point. However, various issues can occur with this synchronization mechanism:

#### Detailed explanation

1. **Uneven workload distribution**:
   - **Problem**: If the workload is unevenly distributed among threads, some threads may finish their tasks much earlier than others and wait idly at the barrier, leading to inefficient use of resources.
   - **Example**: In a parallel algorithm where each thread processes a portion of an array, if some portions take significantly longer to process than others, faster threads will spend time waiting at the barrier.

2. **Straggler effect**:
   - **Problem**: A single slow thread (straggler) can delay the entire batch of threads at the barrier, causing performance degradation.
   - **Example**: In a distributed system, if one node is slower due to network latency or lower processing power, it will cause all other nodes to wait, affecting overall system performance.

3. **Resource contention**:
   - **Problem**: When multiple threads converge on a barrier, there can be contention for the resources managing the barrier (e.g., locks or semaphores), potentially leading to performance bottlenecks.
   - **Example**: If a barrier implementation uses a shared lock, contention for this lock can increase as the number of threads grows, leading to increased wait times.

4. **Deadlock**:
   - **Problem**: Incorrectly implemented barriers can lead to deadlocks, where threads are permanently blocked waiting at the barrier due to a logical error in the synchronization code.
   - **Example**: If a barrier is supposed to synchronize 10 threads but is mistakenly programmed to wait for 11, all threads will wait indefinitely, causing a deadlock.

5. **Livelock**:
   - **Problem**: Although less common, livelock can occur if threads constantly change state in response to each other without making progress through the barrier.
   - **Example**: Threads repeatedly entering and leaving the barrier due to incorrect signaling can lead to livelock, where no thread progresses past the barrier.

6. **Complexity in nested barriers**:
   - **Problem**: In complex programs with nested barriers, ensuring correct synchronization can be challenging and prone to errors, leading to unexpected behavior.
   - **Example**: If an outer barrier depends on the completion of an inner barrier, and there's a misconfiguration, threads may be incorrectly synchronized, causing logic errors.

7. **High overhead**:
   - **Problem**: The overhead associated with managing barriers can become significant, especially in systems with a large number of threads, leading to performance issues.
   - **Example**: The time taken to manage and coordinate the barrier increases with the number of participating threads, reducing the benefits of parallelism.

#### Solutions and best practices

1. **Dynamic load balancing**:
   - Implement dynamic load balancing to ensure more even distribution of work among threads, reducing the likelihood of uneven workload distribution.

2. **Hierarchical barriers**:
   - Use hierarchical barriers that synchronize threads in smaller groups before synchronizing the entire set, reducing contention and overhead.

3. **Timeout mechanisms**:
   - Implement timeout mechanisms to detect and handle situations where threads are waiting too long at a barrier, potentially indicating a problem like a deadlock.

4. **Profiling and optimization**:
   - Profile the application to identify bottlenecks and optimize the code to reduce the time threads spend waiting at barriers.

5. **Efficient barrier implementations**:
   - Use efficient barrier implementations that minimize contention and overhead, such as tree-based barriers or software combining trees.

6. **Graceful degradation**:
   - Design the system to handle stragglers gracefully, allowing other threads to perform useful work while waiting.

By understanding and addressing these issues, developers can design more efficient and robust parallel programs that make effective use of barrier synchronization.

### Atomicity violation

Atomicity violations occur when a sequence of operations that should be executed as a single, indivisible (atomic) operation are interrupted by other threads, leading to inconsistent or incorrect results. This issue is common in concurrent programming, where multiple threads access and modify shared data.

#### Detailed explanation

1. **Definition of atomicity**:
   - **Atomic operation**: An operation or a set of operations that are performed as a single unit without interference from other operations. Either all operations are executed, or none are, ensuring data consistency.

2. **Common scenarios for atomicity violations**:
   - **Check-then-act**: A thread checks a condition and then acts based on the result, but another thread changes the condition in between.
   - **Read-modify-write**: A thread reads a value, modifies it, and writes it back, but another thread modifies the value in between the read and write steps.

3. **Example scenarios**:
   - **Bank account example**:
     - Suppose two threads are transferring money from a shared bank account. The first thread checks the balance to ensure there are sufficient funds before making a transfer, while the second thread is simultaneously transferring money out. Without synchronization, both threads could read the same initial balance and proceed with the transfer, resulting in an overdrawn account.
   - **Counter increment example**:
     - Consider two threads incrementing a shared counter. Both threads read the current value, increment it, and write it back. Without synchronization, both threads might read the same value, increment it, and write back the same result, causing one increment to be lost.

4. **Consequences of atomicity violations**:
   - **Data corruption**: Shared data can become inconsistent or corrupted.
   - **Incorrect program behavior**: The program may produce incorrect results or exhibit unexpected behavior.
   - **Security vulnerabilities**: In some cases, atomicity violations can lead to security issues, such as race conditions exploited by attackers.

5. **Detection and debugging**:
   - **Testing and debugging**: Atomicity violations can be difficult to detect through testing because they may only occur under specific timing conditions. Tools like thread analyzers and race condition detectors can help identify these issues.
   - **Code review**: Careful code review and understanding of concurrent programming principles can help spot potential atomicity violations.

6. **Preventive measures and solutions**:
   - **Locks (mutexes)**:
     - Use locks to ensure that a sequence of operations is executed atomically. For example, acquire a lock before checking and modifying a shared variable and release it afterward.
   - **Atomic operations**:
     - Use atomic operations provided by the programming language or library (e.g., `AtomicInteger` in Java, `std::atomic` in C++) to perform atomic read-modify-write operations.
   - **Transaction memory**:
     - Utilize transactional memory systems where a series of read and write operations are grouped into a transaction, ensuring atomicity.
   - **Higher-level concurrency constructs**:
     - Use higher-level constructs like semaphores, barriers, or synchronized collections that manage synchronization internally to prevent atomicity violations.
   - **Volatile keyword**:
     - In languages like Java, use the `volatile` keyword to ensure visibility and ordering of changes to a variable across threads, though this alone does not ensure atomicity.

7. **Programming practices**:
   - **Minimize shared data**: Reduce the amount of shared data that needs to be accessed concurrently.
   - **Immutable objects**: Use immutable objects to avoid the need for synchronization on shared data.
   - **Design for concurrency**: Design the application with concurrency in mind from the start, considering how threads will interact with shared resources.

## Other problems

### Crossing the river
You have been hired to coordinate people trying to cross a river. There is only a single boat, capable of holding at most three people. The boat will sink if more than three people board it at a time. Each person is modeled as a separate thread, executing the function below:

```cpp
void Person(int index, int location)
// location is either 0 or 1;
// 0 = left bank, 1 = right bank of the river
{
    ArriveAtBoat(index, location);
    BoardBoatAndCrossRiver(location);
    GetOffOfBoat(index, location);
}
```

Synchronization is to be done using monitors and condition variables in the two procedures `ArriveAtBoat` and `GetOffOfBoat`. Provide the code for `ArriveAtBoat` and `GetOffOfBoat`. The `BoardBoatAndCrossRiver` procedure is not of interest in this problem since it has no role in synchronization. `ArriveAtBoat` must not return until it safe for the person to cross the river in the given direction (it must guarantee that the boat will not sink, and that no one will step off the pier into the river when the boat is on the opposite bank). `GetOffOfBoat` is called to indicate that the caller has finished crossing the river; it can take steps to let other people cross the river.

#### Solution

```cpp
class Boat
{
private:
    int _location = 0;

    std::vector<int> _load;
    std::vector<Condition> _conditions;

    const Lock _global_lock = Lock();

public:
    const int CAPACITY = 3;

    Boat()
    {
        for (int i = 0; i < 2; i++)
        {
            _conditions.push_back(Condition(&_global_lock));
        }
    }

    void board(int index, int location)
    {
        _global_lock.acquire();

        while (location != _location || _load.size() == 3)
        {
            _conditions[location].wait();
        }

        std::stringstream ss;
        ss << "Person " << index << " boarded: " << location << " -> " << 1 - location << "\n";
        std::cout << ss.str();

        _load.push_back(index);
        _global_lock.release();
    }

    void get_off(int index, int location)
    {
        _global_lock.acquire();

        std::stringstream ss;
        ss << "Person " << index << " got off: " << location << " -> " << 1 - location << "\n";
        std::cout << ss.str();

        _load.pop_back();
        if (_load.empty())
        {
            _location = 1 - _location;
            _conditions[_location].notify_all();
        }

        _global_lock.release();
    }
};

const int PEOPLE_COUNT = 4;
Boat boat;

void ArriveAtBoat(int index, int location)
{
    boat.board(index, location);
}

void GetOffOfBoat(int index, int location)
{
    boat.get_off(index, location);
}

void BoardBoatAndCrossRiver(int location)
{
}

void Person(int index, int location)
// location is either 0 or 1;
// 0 = left bank, 1 = right bank of the river
{
    ArriveAtBoat(index, location);
    BoardBoatAndCrossRiver(location);
    GetOffOfBoat(index, location);
}

DWORD WINAPI routine(void *_index)
{
    int index = *(int *)_index, location = 0;
    while (true)
    {
        std::stringstream ss;
        ss << "Person " << index << " at " << location << "\n";
        std::cout << ss.str();

        Person(index, location);
        location = 1 - location;
    }
}

int main()
{
    std::vector<HANDLE> threads;
    for (int i = 0; i < PEOPLE_COUNT; i++)
    {
        threads.push_back(CreateThread(NULL, 0, &routine, new int(i), 0, NULL)); // ignore memory leak
    }

    WaitForMultipleObjects(threads.size(), &threads[0], TRUE, INFINITE);
    return 0;
}
```

Note that this solution may lead to starvation. The readers is encouraged to develop a starvation-free solution. 

## References

- https://en.wikipedia.org/wiki/ABA_problem
- https://pub.dev/documentation/async_locks/latest/async_locks/async_locks-library.html
- https://docs.python.org/3/library/threading.html
- https://en.wikipedia.org/wiki/Sleeping_barber_problem
- https://en.wikipedia.org/wiki/Cigarette_smokers_problem
- https://www.cis.upenn.edu/~devietti/papers/lucia.atomaid.toppicks.2009.pdf
- https://www.cs.cornell.edu/courses/cs4410/2010fa/synchreview.pdf
