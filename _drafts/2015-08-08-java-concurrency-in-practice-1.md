## Chapter2. Thread Safety

### State, state, state

> Writing thread-safe code is, at its core, about managing access to state, and in particular to shared, mutable state.

synchronization mechanism in Java

* `synchronized` keyword
* volatile variables
* explicit locks
* atomic variables

### thread-safe

> a class is thread safe when it continues to behave correctly when accessed from multiple threads.
> 


### synchronized 

> A synchronized method is a shorthand for a synchronized block that spans an entire method body, and whose lock is the object on which the method is being invoked. 

> Every Java object can implicitly act as a lock for purposes of synchronization; these built-in locks are called intrinsic locks or monitor locks.

> Intrinsic locks are reentrant if a thread tries to acquire a lock that it already holds, the request succeeds.

> If that same thread acquires the lock again, the count is incremented, and when the owning thread exits the synchronized block, the count is decremented. When the count reaches zero, the lock is released. 

### volatile

> Locking is not just about mutual exclusion; it is also about memory visibility. 

> Locking can guarantee both visibility and atomicity; volatile variables can only guarantee visibility.

### Safe construnction 

> When an object creates a thread from its constructor, it almost always shares its `this` reference with the new thread, either explicity (by passing it to the constructor) or implicitly (because the Thread or Runnable is an inner class of the owning object). 

### ThreadLocal

> allows you to associate a per-thread value with a value-holding object

### Immutable

> There is a difference between an object being immutable and the reference to it being immutable.


### Static initializer

> Static initializers are executed by the JVM at class initialization time; because of internal synchronization in the JVM, this mechanism is guaranteed to safely publish any objects intiailized in this way.

