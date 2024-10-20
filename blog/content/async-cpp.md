+++
title = "Asynchronous Multi-Thread Design Pattern with C++"
date = 2024-10-19
draft = false
[taxonomies]
  tags = ["Design Pattern", "C++"]
[extra]
  toc = true
	keywords = "Async, Multithread, C++"
+++

This post does NOT talk about existing async frameworks in C++.

We will talk the basic concept of asynchronous multi-thread design pattern in C++ and provide a simple example program.

In C++ there is no built-in language support for something like asynchronous channels in Rust. And C++ beginners often find themselves lost to plan their multithread program.

## Asynchronous Design Pattern

In traditional synchronous programming, the parent and child threads are usually interlocked with some kind of synchronization mechanism, such as a state machine. The parent the child threads share access to the same memory for communication.

The problem of synchronous pattern is, you need to program the synchronization steps very carefully, otherwise there will be data racing and deadlock situations. Moreover, for a growing software, the sychronization complexity grows exponentially when new steps are added. It has serious scalability issues, so most modern system already ditched this design pattern in their software.

The asynchronous pattern requires the parent not to interrupt the child thread at all, and only interact with the child threads with given I/Os, typical working process is：

* Parent thread sends a task data structure to one or a group of child threads.
* A child thread unblocks upon receiving task data and starts running the task.
* Child thread sends the result back to parent thread and waiting for the next task.
* After parent thread finishs dealing with other tasks, it blocks and waits for the result it asked a moment ago.

You may wonder where are the synchronization steps? Synchronization is done by the parent thread only, although you can add states to the child thread as well, it is generally a bad practice because these steps add unnecessary complexity.

Typically parent and child threads communicate through FIFO queues so the data can be temporarily stored in the queue while the reader is busy doing something else.

## Example C++ Program

Here is an example program, the main thread want to distribute the math calculations to a group of worker threads (assume the calculation is just `square()` function).

We can first define the FIFO queue as the following structure:

```cpp
template <class T>
struct WorkerChannel_T {
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
    // Thread safe
    void push(T data) {
        std::unique_lock lk(mutex_);
        queue_.push(data);
    }
    // Thread safe
    T pop() {
        std::unique_lock lk(mutex_);
        cv_.wait(lk, [this] { return !this->queue_.empty(); });
        T result = queue_.front();
        queue_.pop();
        return result;
    }
    void notify() { cv_.notify_all(); }
}
```

Notice that `push()` and `pop()` are both protected by the `mutex_` so it can be used by both parent and child threads.

Now let's define the worker thread as:

```cpp
void square_worker(WorkerChannel_T<float>& in_ch,
                   WorkerChannel_T<float>& out_ch) {
    while (true) {
        float in_num = in_ch.pop();
        if (std::isnan(in_num)) {
            break;  // End thread
        } else {
            in_num = std::pow(in_num, 2);
            // Simulate 100ms time consumption
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            out_ch.push(in_num);
            out_ch.notify();
        }
    }
```

Now for our main thread we can simply:

```cpp
int main() {
    WorkerChannel_T<float> in_ch;
    WorkerChannel_T<float> out_ch;
    auto worker_vec = std::vector<std::thread>();
    // Create 8 worker threads
    for (int i = 0; i < 8; i++) {
        worker_vec.push_back(
            std::thread(square_worker, std::ref(in_ch), std::ref(out_ch)));
    }
    // Post tasks
    for (int i = 0; i < 100; i++) {
        std::cout << "Post number: " << float(i) << std::endl;
        in_ch.push(i);
    }
    // Notify CV
    in_ch.notify();
    for (int i = 0; i < 100; i++) {
        float j = out_ch.pop();
        std::cout << "Get result: " << float(j) << std::endl;
    }
    // Stop all worker threads
    for (int i = 0; i < worker_vec.size(); i++) {
        in_ch.push(NAN);
    }
    in_ch.notify();
    // Join all thread when we done
    for (int i = 0; i < worker_vec.size(); i++) {
        std::cout << "Join worker[" << i << "]" << std::endl;
        worker_vec[i].join();
    }
    return 0;
}
```

We can share the same `WorkerChannel_T<float>` with more than one workers. Since only one of them can get the lock and `pop()` a task from the queue. Also the output channel is shared between them as well.

You can find the whole example C++ code on [Godbolt.org](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAMxcpK4AMngMmAByPgBGmMQgAGyJpAAOqAqETgwe3r4BQemZjgJhEdEscQnJtpj2JQxCBEzEBLk%2BfoG19dlNLQRlUbHxSSkKza3t%2BV3j/YMVVaMAlLaoXsTI7Bzm/uHI3lgA1Cb%2BbsgsTAQIJ9gmGgCCO3sHmMenaAz4DQD6AG4teCYMXoNzujzMuwY%2By8RxObic42ImFYoIeTyhLzebhYXgImFUqPBkOhsNOAEcvJhKYT0STXnCrkimOgaRDnjD6adfpgHCQaQ88SxUgY8Vj9kwFApDgAVQmIrwOQ4AdRIAGt4m4EIYIrRvtLjgB2KwPQ6mw7jdAgEAUqmYOGy/zYQ42ynfE7G%2B5m80ES0gHF41SHf34t3%2BD1ei1Wj5fbJ/AFA%2BiHZC/UPhs0AenTMoQTPQ5qYVDtJrNv1QeDzqS8CgQEH16EuTCWhrTXtNkZAXgYeBt31oqGQqsOtFVEGDqm%2BS3dYNbZpdmG%2BADpK9WIPXmpOw9OzSYDQARLemzPZ3P5wsHmWHdKpCBNnct1vtzvd119gdDkdjidT4sz5OLgDuTCEBAw6kMcACsFhXHgCgmOBu7NocSIEOsDCHGAYDQQoAC0NxzoumBCgQACeN7uoau4bveXr6kiCheLQBBvAh%2BELlQxCyGRm4/q2rFXlx1FmshqFIZg9GMd%2Bnrbnu56luWhwMKgjhUKRt5GkmKYLopynEd8Yi0AJFFgju%2B7cY8DxyXmCgUi087/mq8QQCqxDqsQmranUepwlQfaXKCZiJIc4TfMgCCkOeM6Raazmue5DA6l5pw%2BagfmOuYgVrAQIUIGp97/ggdCvBABDEJSuURWayWXEFDDfAwPjMTV2VLqg15URVpp4FQhwQO2MHMAwEDBfVLBLOVPFRTETKquRhxHtgnyHIyyIshN0kIXUSjNh1rbDQ1JwIe26T/kNtUjWBZjtWtM5HkIeA4iKrxcBoGgsFKjhsEmAj0UR2Q7RGPpWlh3zLcyVoKPQmCpN8/DEL1gMoDmAioFaLB0LQME8gI6AKBAz0aGNklRV6mXNcuNZ7aNRPE6apOhVpSldap1MziZ55s2iMlog84RMRc4Q3ttPExRqWrxZ5DpuFVBA3E1oUs8qDluWLCWS9Lst09cZlekwuKoIc9kufEfw8o17bcryxBwu2oMso6AnnkebhMqKAAcBtK0tOYrbBPGwz1vNBY1GjkXgWKHK7ofWNY41STOhuuSbyBLlWCDfDETADhA/1mjb3vMr1NlIt8CfxGB7ZIlQp3ZSs3q%2BpXECa2NV1x0ZPFHsoGRMc0Ciqr7rf%2B6dTFhwdhwh2GQdwoc%2BNR5YMdC63AO%2BmguJYlP5hmJ34wKcMCTHGYZhr6chzS0Nt6nFP7auLQCtesF9Pk2fCsc63R6RIzKmHG4ABq573wgDMdIOz9iQAOghJ7%2BAQuPCwEC3DTxerPKwlhY5RWlocbQjVNYtTarfXOCMV5MThOvA%2BABxTATE6IMQICAfeh8iHH1Ptoc%2BbhL4I2vs/Lmr8sxNFaocfSHsjbEC9rmfuXpB6BxHpAseodw6l2IEnBcmQABemBDJ4GjsgheUV/4pxXJEe4kQW5ehfnfWq9NtJM2AVww4AApMsaF%2BG2wNggRgBtXjoAEEWAeoCh4QKgTIqeciFHKNUVRIKGiLAoMiu2AhR84EbzseEARrk4JILoRfY%2BkiWHHw3nBfcB84l1ytOw7WkUgkW1Seo%2BCC5tD2KscYzhXphLEDQtA4ye4OArFoJwcCvA/DcF4CjDgLC56WHNGsDY9IIQ8FINQjgWgxqkFVAEfwC5/DrI2ZszZKRukcEkH0zQgzOC8AUCADQszDkrDgLAJAaAhSFTIBQCAdzUgPJAMAKQZg%2BB0DxMQU5EAYiHNIDEcILRiKcBmSC5gxBiIAHkYjaB5HMmZdy2CCFhQwWg4L5m8CwDELwwA3D6VOQM0gWALhGHEDisleAkQODwNyElWhgiqB5LiLYzLeZ1CBRjKaYKPBYCBSVe6ELeDcmIDEDImBdyEUMMADGRhLl8AMMABQ388CYH/LC1IjBRUyEECIMQ7ApD6vkEoNQQLdBBAMIq0woybC8tOZAFYrUGgksGeK4g5YxLwAgMwNgIALTitIP8bw7A7VIIsGYAmKw7BIuyC4T4Uw/BBFCOEIYlQRiFAyFkAQya9BFFzQweYu89BxvpQIPokxPAdDLXUeNlaJgDHTQsLNtgm35qCLMVoJbM0JC4LGiZmwJBdJ6Qc6lQzDiqFdokbCiRJCHGAMgZA09JALkPhAXAhBQE7AHbwOZCyVjLK4OBBckgACcGgzCu2jZIF6d7z1cANPoTg%2BzSD9OZUMk5ZyLk4quTARAIBMqVgIOQSgLyHmRFYFsads752LuXau9dvBMBfBIOWPQ/ADWiHECarDZqVDqGpVa0g/5iBMFSKK0dHBenvqBUM2FuIQOHFQN1WDc6F1LpXVIddPUPD3PoEI3dSx92XKPSAcCXAFxPsSP4V2khEgBUU%2BBA00hdlvo/Ucjg37zkHs6dRsw47P3HN/YekN8RMjOEkEAA).

If you are familar with the other async frameworks you will notice that:

* The `in_ch.push()` then `in_ch.notify()` combination is actually same as you create a `promise` or `future` using the async framework.
* The `out_ch.pop()` is actually same as `await`. The parent thread blocks on calling this function to wait the results became available.

We just create a tiny framework in a few lines with `std::thread`! However the modern async framework usually is more complicate where the executioner could be a thread pool or co-routines instead of the system threads. However the concept displayed is the same.

## One Step Forward

Just like shown in the above example, **you can share a FIFO channel among multiple workers**. Because we use a mutex to protect the queue, it is safe for all workers to block on one channel. 

It is relatively easy to design the worker thread, since it typically only needs to work on a single I/O pair.

However you may also notice that the parent thread can only be blocked on the single channel, what if there are multiple channels that the parent thread needs to watch simultaneously?

### Serialized Wait

This the most straight forward answer, that is the parent just wait the result one by one.

It may sound stupid, it is acutally very useful. The parent acutally only needs to wait until the slowest child finishes its task at most.

```cpp
result1=out1_ch.pop();
result2=out2_ch.pop();
result3=out3_ch.pop();
result4=out4_ch.pop();
```

After serialized wait like shown above is finished, the parent is guranteed to have all the data ready for the next step.

This situation is usually used when the parent thread needs to have a set of data ready before moving on.

### Wait until the first comes

There is another scenario that we want the parent thread to wait until the first result is availble.

The easiest solution is actually quite simple, we just need to use `std::variant` and share the same channel (with `std::variant` as storage type) among different threads. For parent thread we can use `std::visit` to process the result further.

However the easiest solution is static, Because you cannot change the channel during the runtime.

You can also have a dedicated channel. When a child pushes new result, the child uses this channel to report which channel number is ready for parent thread to read.

