# cpp_concurrency

Code for the Modern C++ Concurrency in Depth course from Udemy. 

## Resources

- https://en.cppreference.com/w/cpp/thread/thread
- https://en.cppreference.com/w/cpp/utility/functional/ref
- Course source code: https://github.com/kasunindikaliyanage/cpp_concurrency_masterclass

## TODO

- jthread
- futures
- latches and barriers
- C++20 addons

## Building the code

```bash
sudo apt install gcc-10 g++-10 # C++20

mkdir -p build && cd build
cmake .. && make && src/section_1/01_joinability

#cmake -D CMAKE_C_COMPILER=gcc-10 -D CMAKE_CXX_COMPILER=g++-10 .. && make
#-std=c++20 -fcoroutines -pthread
```

## C++ Thread Support Library

- [Basic Concepts](#basic-concepts).
- [Thread Management](#thread-management).
- [Locking Mechanisms](#locking-mechanisms).
- [Condition Variables and Futures](#condition-variables-and-futures).

### Basic Concepts

**Process:** Is defined by its instructions and current state.

**Context Switching:**
- The OS can store the state of a process or thread, so that it can be restored and resume execution at a later point. 
- This allows multiple processes to share a single central processing unit (CPU), and is an essential feature of a multitasking operating system.

**Round-Robin Scheduling:**
- Assigns time-slots to each process in equal portions and circular order, having no priorities.
- Context switching is performed when a time slot is completed.
- Simple and starvation-free.
- See: https://en.wikipedia.org/wiki/Round-robin_scheduling

**Task- and Data- Level Parallelism:**
- Task-Level: Threads perform different tasks (using the same or different data).
- Data-Level: Threads perform the same task on different data.

**Parallelism vs Concurrency:**
- Parallelism: When each process/thread runs on a dedicated processor at the same time.
- Concurrency: Emulated parallelism based on context-switching (sequencial execution!).
- True parallelism is difficult to achieve, due to the low number of cores usually available.

**Heterogeneous Computing:**
- Refers to systems that use more than one kind of processor or cores. 
- These systems gain performance or energy efficiency by using specialized hardware.
- https://en.wikipedia.org/wiki/Heterogeneous_computing

**GPGPU (General Purpose GPU):**
- Is the use of a GPU to handle tasks a CPU would do.
- The idea is to handle matrix-like data using the GPU.
- See: https://en.wikipedia.org/wiki/General-purpose_computing_on_graphics_processing_units


### Thread Management

**Thread Object**: A properly constructed thread object represents an active thread of execution in hardware level.

**Joinability**:
- *Properly constructed* thread objects are *joinable*. Default constructed are *non-joinable*!.
- Calling the `join` or `detach` makes the object *non-joinable*.
- If `join` or `detach` are not called, then `std::terminate` will be called by the destructor.
- Programs having calls to `std::terminate` (aborts program) are referred as *unsafe*.
- See: [example](src/section_1/01_joinability.cpp).

**Join Function** [std::thread::join()](https://en.cppreference.com/w/cpp/thread/thread/join):
- Introduces a *synchronization point* between the thread and the caller.
- It blocks the execution of the caller, until the thread execution finishes.

**Detach function** [std::thread::detach()](https://en.cppreference.com/w/cpp/thread/thread/detach):
- Separates the thread from the thread object, allowing the thread to continue independenly.
- Detached threads may outlive parents.
- Detached threads are *non-joinable*, thus they can be safely destroyed.
- Detached threads usually cannot outlive the `main()` function, as the OS terminates all threads on exit [stackoverflow.com](https://stackoverflow.com/questions/19744250/what-happens-to-a-detached-thread-when-main-exits).
- Allocated resources will be freed once the thread finishes.
- Be wary of passing by reference to detached threads!. When owner finishes, reference will dangle.
- See: [example](src/section_1/02_detach.cpp).

**Delayed Joins and Exceptions**:
- We might need to delay the call to `join` to not block the thread so early. Notice that we can call `detach` as soon as we want!.
- If an exception is thrown before `join`, `std::terminate` might be called!.
- RAII can be used to handle the thread resource and join when necessary.
- See: [example](src/section_1/03_exceptions.cpp).

**Thread Constructor** [std::thread::thread()](https://en.cppreference.com/w/cpp/thread/thread/thread):
- Default creates non-joinable thread.
- Move enabled and Copy Disabled (see: [example](src/section_1/06_ownership.cpp)).
- Class/Fn arguments are forwarded. Use `std::ref` to pass by reference. (see: [example](src/section_1/04_thread_parameters.cpp), [example](src/section_1/05_pass_by_ref_and_detach.cpp)).

**Useful API** ([example](src/section_1/07_useful_api.cpp)):
- [std::thread::get_id()](https://en.cppreference.com/w/cpp/thread/thread/get_id): Unique id for *active* threads, contant otherwise. The type `std::thread::id` is designed to be used as a key in associative containers.
- [std::this_thread::sleep_for()](https://en.cppreference.com/w/cpp/thread/sleep_for): Blocks execution for *at least* the specified duration. It may block longer due to scheduling or resource contention delays.
- [std::this_thread::yield()](https://en.cppreference.com/w/cpp/thread/yield): Hints the scheduler to allow other threads to run, and re-inserts the thread into the scheduling queue.
- [std::thread::hardware_concurrency()](https://en.cppreference.com/w/cpp/thread/thread/hardware_concurrency): Returns the number of concurrent threads supported by the implementation (logical cores). The value should be considered only a hint.
- [thread_local](https://en.cppreference.com/w/c/thread/thread_local): Macro specifying that a variable has thread-local storage duration; Each thread has its own, distinct, object. Initialization and destruction are bound to the thread.


**More Examples**:
- Exercise 1: [code](src/section_1/exercise_1.cpp)
- Exercise 2: [code](src/section_1/exercise_2.cpp)
- Exercise 3: [code](src/section_1/exercise_3.cpp)
- Parallel Accumulate: [code](src/section_1/08_parallel_accumulate.cpp).

### Locking Mechanisms

The most common problem in multithreading implementations are broken invariants while updating data.

**Race Condition**: When the outcome depends on the sequence or timing of operations on the threads. They are hard to reproduce and debug, as the operation is non-deterministic and any observer may hide the problem.

**Data Race**: It is a type of *race condition* where memory operations in multiple threads attempt to access/modify the same location simultaneously. These code sections are referred as *critical sections*, they should be *mutually exclusive*. It is posible to have non-deterministic output even if the program is data-race free.

**Mutex**: Provides mutual exclusive access to shared data from multiple threads. However, adding a mutex lock to every function will result in no parallelism at all.

**Locks Bypassing**: Is dangerous and it is based on allowing other to keep a reference to the raw data. Then, we cannot guarantee proper locking, even if locks are used internally ([examples](src/section_2/02_mutex_misusages.cpp)):
1. If the container provides a function to get a reference or pointer to the resource.
2. If the container provides a function to run any code over the data.

**Race Condition Inherited from the Interface**: Having all methods guarded by a lock guarantees thread safety at the function execution level. However, race conditions can still happen if the user code depends on combinations of those functions. E.g., when `empty()` is used to later do `pop()` on a container ([example](src/section_2/03_thread_safe_stack.cpp)).

**Deadlock**: Is a state in which each thread is halted waiting for a lock which won't be released. It happens when multiple locks required, and they are not acquired in the same order. A deadlock without using locks!, by having threads attempting to call `join()` on the others. In the [example](src/section_2/03_thread_safe_stack.cpp), two scenarios arise:
1. The `transfer()` function attempts holding 2 locks for different accounts. But each account locks its own mutex first.
2. Different functions attempt holding 2 locks in different order.

**C++ Mutex and Guards**:
- The [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) primitive implements a mutex, for which we must call `lock()` and `unlock()`.
- The [std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard) is a RAII-style mutex wrapper for managing the lock automatically.
- The [std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock) is similar to `std::lock_guard`, but has a variadic constructor taking more than one mutex. This allows to lock multiple mutexes in a deadlock avoiding way as if `std::lock` were used. See also [stackoverflow.com](https://stackoverflow.com/questions/43019598/stdlock-guard-or-stdscoped-lock).
- The [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) is similar to `std::lock_guard`, but it does not have to acquire the lock during construction. It also allows time-constrained locking, recursive locking, conditional locking, and ownership transfer. In particular, the lock deferral allows acquiring multiple locks later using the `std::lock` function, as if `std::scoped_lock` were used.
- [mutex, lock_guard, and scoped_lock examples](src/section_2/01_mutex.cpp), [unique_lock examples](src/section_2/05_unique_lock.cpp).

**STL and Thread Safety** [STL container](https://en.cppreference.com/w/cpp/container):
1. All container functions can be called concurrently by different threads on different containers.
2. All `const` member functions can be called concurrently by different threads on the same container. Operations like `begin()`, `end()`, `rbegin()`, `rend()`, `front()`, `back()`, `data()`, `find()`, `lower_bound()`, `upper_bound()`, `equal_range()`, `at()`, and, except in associative containers, `operator[]`, behave as const for the purposes of thread safety.
3. Different elements in the same container can be modified concurrently by different threads, except for the elements of `std::vector<bool>`.
4. Iterator operations (e.g. incrementing an iterator) read, but do not modify the underlying container, and may be executed concurrently with operations on other iterators on the same container, with the const member functions, or reads from the elements. Container operations that invalidate any iterators modify the container and cannot be executed concurrently with any operations on existing iterators even if those iterators are not invalidated.
5. Elements of the same container can be modified concurrently with those member functions that are not specified to access these elements.
6. In any case, container operations (or any other STL functions) may be parallelized internally as long as this does not change the user-visible results.

### Condition Variables and Futures

**Condition Variables**:  [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable) is a synchronization primitive used to block one or multiple threads until another thread modifies the variable and notifies to it:
- The notifier thread has to acquire a mutex to modify the variable (even if it is atomic). Then, it must execute `notify_one` or `notify_all` (no lock needed).
- Waiting threads must acquire the same mutex and:
  1. Check the condition. Execute `wait`, `wait_for`, or `wait_until`, atomically releasing the mutex. And awake on timeout, notification, or [spurious wakeup](https://en.wikipedia.org/wiki/Spurious_wakeup). Finally the condition must be checked to continue waiting or resume if needed.
  2. Or just use the predicated overload of `wait`, `wait_for`, and `wait_until`.
- For maximum efficiency, `std::condition_variable` works only with `std::unique_lock<std::mutex>`, while `std::condition_variable_any` works only with any lock.
- [example](src/section_3/01_condition_variable.cpp).
