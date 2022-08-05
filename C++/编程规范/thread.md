```cpp
inline void Condition::Signal()
{
    /*
     * POSIX says pthread_cond_signal wakes up "one or more" waiting threads.
     * However bionic follows the glibc guarantee which wakes up "exactly one"
     * waiting thread.
     * man 3 pthread_cond_signal
     * pthread_cond_signal restarts one of the threads that are waiting on
     * the condition variable cond. If no threads are waiting on cond,
     * nothing happens. If several threads are waiting on cond, exactly one
     * is restarted, but it is not specified which.
     */
    pthread_cond_signal(&mCond);
}
```
