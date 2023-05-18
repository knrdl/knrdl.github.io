---
title: "Multiprocessing, Multithreading, Asynchronous programming: What's the difference?"
date: "2023-05-18"
tags: [programming, async, python, nodejs, rust, go, javascript]
lang: "en"
---

# The problem

In programming there is often a need to run things in parallel, for example:
* Execution time is optimized if tasks run in parallel
* A webserver needs to handle multiple requests simultaneously
* Sending an email in the background should not block the UI


There are three different approaches for these requirements with structural differences.

# Multiprocessing / Multiprogramming

Multiprocessing is the oldest attempt of parallelization. It is useful for batch processing with no dependencies between the jobs. Typical use cases are:
* converting a lot of images between formats (converting image #1 has no impact on the conversion of image #2)
* compressing files individually
* converting each slide of a presentation into the PDF format and concat them afterward into a single PDF file

A typical setup looks like this:
- one **manager** process: keeps a task queue and coordinates the workers by distributing the tasks among them
- one **worker** process per logical CPU core: performs the supplied task and gets the next one from the manager when finished

Using one worker process per processor core is most of the time reasonable. As each job runs on a processor core the full processing capabilities of the machine are utilized. But there is still the coordination overhead of the manager process which requires context switches by the operating system. If the machine for example has 4 CPU cores there will never be a 4x speedup by embracing multiprocessing instead of running the jobs one after another. Instead, maybe a 3.8x speedup is more likely to achieve. This also depends on the specific kind of workload. Practically speaking the operating system also has to handle other processes as well which also consume CPU time.

But it is still possible to reduce the management overhead of the workers. Instead of supplying single tasks to the workers, they can be fed with small batches of tasks.

This concept is useful when there is no shared memory between the tasks, e.g. converting image #170 has no influence on the conversion of image #171 or #172. A typical implementation might use the process infrastructure as provided by the operating system. For an application developer this is easy to implement but spawning new processes has an overhead on its own. That's where multithreading comes into play.


# Multithreading

Multiprocessing works fine as long as there is no shared memory between tasks. But what if two tasks need to communicate with each other to fulfill their goal:
* converting a video in the backend might need to report real-time progress to the frontend
* a database connection might be used by multiple requests
* a server handling HTTP requests might keep a global state which affects the processing of following HTTP requests
* automatic sending an email might fail but can be retried. To limit the number of retries a failure counter might be kept in the database and is incremented until the upper bound is reached. Meanwhile, the execution of the main server program should continue flawless.

So multithreading is always applicable if multiple components of an application should run at the same time and must be able to communicate with each other by sharing in-memory state. Sharing state is what differs multiprocessing from multithreading. In multiprocessing environments grabbing state is normally only wanted after the execution of a job. For example the manager might check the console output of a worker to decide if a failed job should run again or execution is given up and the job is marked as finally failed. 

In contradistinction to multiprocessing multithreading allows real-time information exchange between the threads of an application. Each process can spawn multiple threads and their threads can also spawn new threads on their own. As threads are regularly also handled by the operating system they are sometimes referred to as "lightweight processes". But an application can also implement a threading concept on its own besides the threading capabilities provided by the operating system. These application handled threads are often called "green threads".

Multithreading has its benefits: parallelized shared in-memory state, parallel execution inside an application, spawning threads is cheaper than spawning processes. But this architecture brings in a new class of problems: race conditions.
If two threads can access the same memory there will be corruptions sooner or later. These bugs are hard to debug. For example thread #1 might be writing a string variable while thread #2 is reading it out. The result will be that thread #2 is reading a malformed string which contains old and new parts. As this error is not fatal the application will work on with corrupted data. This can evolve to a security problem (e.g. TOCTOU) or corrupt data retention integrity. Therefore, a bulletproof locking mechanism is compulsory when working with multithreading. For example thread #1 sets a mutex flag before writing to the variable. Now thread #2 waits until the mutex is unset by thread #1 before reading out the value. There can still be problems. For example the programmer might forget to unset the mutex so thread #2 waits until infinity before going on. As the rest of the application (the other threads) stay operational this class of bugs is also hard to detect. Therefore, it is a good idea to use atomic data types (as found e.g. in c++, go, rust) which handle locking on their own. 

However, locking as mitigation technique might still produce problems like deadlocks: E.g. mutex #1 is set and waits for mutex #2 to be unset. But if mutex #2 waits for mutex #1 to be unset the situation becomes unsolvable without giving up the locks. To prevent locking from being essential part of the programming experience some high level languages have taken workarounds, for example JavaScript and Python.

In the past web browsers have used a single thread to render an open tab. That's why JavaScript is designed to be a single-threaded language. There is just no native threading concept in JavaScript. Nowadays extensions to the web standards like the [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) have introduced the possibility to do multithreading with JavaScript. However, JavaScript per default runs single-threaded and strongly utilizes async programming instead, which is explained in the section below.

In contradistinction to JavaScript Python is designed with multithreading in mind but wants to avoid the pitfalls of application-handled locking. Therefore, the Python interpreter implements a technique called global interpreter lock (GIL). A Python application can spawn multiple threads with no trouble. However, these threads never run at the exact same time even if it looks that way to the application written in Python. Instead, the GIL schedule the execution of only one thread at a time. This pseudo-multithreading discharges the developer from thinking about race conditions. Two threads accessing the same variable will simply never run at the same time in realtime. So all thread crossing operations are atomic by default. But on the other hand, because it is just a pseudo-multithreading there is no performance gain in parallelizing workloads via threads in Python. The GIL might be relieved in future Python versions but for now there are 3 alternatives:
1. Use multiprocessing instead
2. Use async programming instead, see section below
3. Write the performance critical path of the application as library in another language and use a Python wrapper to access it from Python code

Option 3 sounds complicated but is actually straightforward. Python can access libraries written in a variety of languages, e.g. C, C++, Fortran, Rust, Go, Cython. These libraries are not subject to the GIL and must handle locking on their own. For example the [PyCa](https://cryptography.io/en/latest/) project provides cryptographic functions for Python but is written in Rust. So an encryption job done by PyCa in a separate thread can really run at the same time as the rest of the Python application.


# Asynchronous Programming

Multiprocessing and multithreading are efficient when it comes to parallelizing cpu-bound tasks which require a lot of computation. However, in modern programs there is often also a lot of waiting for external resources included. These tasks can be classified as io-bound. Handling these require an additional (the newest of the 3 concepts) paradigm: async programming. A modern server application has to wait for many things like:
* Wait for the user to provide input
* Wait for a file to be loaded (read) from the file system
* Wait for the database to respond to a request
* Wait for the network to download a file
* Wait for an email to be sent
* Wait for another process to finish
* Wait for an operating system syscall, e.g. flush caches
* Wait a specified time, alias sleep

These operations could be modelled as a thread for each operation. But this would be a highly inefficient use of resources: For example a thread would send out an HTTP request and be blocked until the response arrives. In the meantime the thread could not do anything useful (just waiting). But the operating system still has to run the thread. It does so in the same manner it schedules the execution times for all other threads and processes on the machine. This means the operating system will waste cpu-time by waiting for an io-bound task (the HTTP request). That's why multiprocessing/-threading should not be used for io-bound tasks.

Instead of threads a new form of execution is required: the event loop. Async jobs are executed inside an event loop as a queue. When async job #1 has to wait for something the event loop runs another async job #2 in the event loop meanwhile. Execution of async job #1 will continue when it is done with waiting. In the meantime it does not waste CPU resources which will be used to run other async jobs instead. 

JavaScript historically uses async programming instead of multithreading because websites often didn't have to compute much but wait a lot (e.g. for AJAX network requests). This has some interesting side effects like the following, strange-looking JavaScript statement:
```javascript
function myfunc() {
    ...
}
setTimeout(myfunc, 0)  // schedule execution for in 0ms
```
One might say that it could be written shorter as `myfunc()`. But `myfunc()` would execute `myfunc` immediately as part of the current event loop iteration. Instead, `setTimeout(myfunc, 0)` will execute `myfunc` in the next event loop iteration. This hack ensures that all pending computations are done when `myfunc` gets invoked.

Async programming is often introduced by the keywords `async` and `await`. In this example a web server invokes a function `get_user_info` to answer an HTTP request:
```python
async def load_user_from_db(userid):
    ... # load user record from database

async def load_user_profile_photo(userid):
    ... # send out http request to query the profile photo

async def get_user_info(userid):
    job1 = load_user_from_db(userid)
    job2 = load_user_profile_photo(userid)
    db_record = await job1
    profile_photo = await job2
    return {'info': db_record, 'photo': profile_photo}
```
This little pseudocode will asynchronously load information from two sources at the same time. The total execution time is just the time for the longest running job (either `job1` or `job2`) plus the overhead of putting the jobs into the event loop. This is an example of io-bound parallelization. Programmers often get this wrong and write not parallelized code instead:
```python
async def get_user_info(userid):  # bad implementation without async benefits
    db_record = await load_user_from_db(userid)
    profile_photo = await load_user_profile_photo(userid)
    return {'info': db_record, 'photo': profile_photo}
```


In practice tasks are rarely just cpu-bound or io-bound but a mix of both. Event loops are good for io-bound tasks but bad for cpu-bound: A cpu-bound task would keep an event loop busy, so it cannot handle other async jobs. The event loop gets blocked and execution might hang. That's why an async task should free up the event loop as fast as it can. Ideally the async task would only do the waiting part inside the event loop. A heavy computation could be outsourced into a separate thread to relief the event loop. To realize this there would be two threads: #1 is executing the event loop and #2 is executing the heavy computation. Thereby the async job becomes all about awaiting the finishing of thread #2. That's exactly what [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) are made for in client-side JavaScript (websites) or [Worker Threads](https://nodejs.org/api/worker_threads.html) in server-side JavaScript (node.js). 

However, as stated above Python only has pseudo-multithreading and node.js did not have Worker Threads in the past. Therefore, thread-based outsourcing could not help. When the event loop cannot be freed the other attempt is to run multiple event loops. This is achieved by combining multiprocessing with async programming: The manager process spawns multiple worker processes and each worker process runs an independent event loop. The manager will then distribute the tasks among the workers, so no event loop gets fed up. For web servers this is often done with [unicorn](https://rubygems.org/gems/unicorn/) (Ruby), [gunicorn](https://gunicorn.org/) (Python) or [pm2](https://www.npmjs.com/package/pm2) (node.js). However, as multiprocessing is involved there is no shared state between the event loops. This might produce unexpected problems when an application is developed in single worker mode and later operated with multiple workers in production as each worker has its separate state. The solution is to introduce a shared state in form of an in-memory database, e.g. a key-value store like Redis, dragonfly or keydb. This extra growth in architectural complexity can be significant.

A better approach to sharing state would be to leave out multiprocessing at all and combine multithreading with async programming instead. That's exactly what the Go language did with their go-routines. On startup a Go program spawns a thread for each CPU core of the machine. Each thread then starts an event loop inside, called runtime scheduler. The go-routines then use go-channels to safely exchange state between thread borders. 

As go-routines and go-channels are building blocks of the Go language the whole ecosystem of libraries is crafted upon them. That's important because mixing synchronous with asynchronous code is in general problematic. For example, Python started as a synchronous-only language but later added async support. So now there are two ways to do, let's say, a DNS query. Running a sync DNS query in a sync program works fine, same as an async DNS query in an async program. Running an async DNS query in a sync program doesn't make sense: an event loop would need to be started first, execute the DNS query and then be destroyed. Just running a sync DNS query would be cheaper in this setup. But running a sync DNS query in an async program is really dangerous: it would block the whole event loop just like an expensive computation would do. But instead of really doing something acceptable the sync DNS query would block the event loop just to wait for a response. This aspect makes it even harder to get async programming in Python right because you always must make sure to use async libraries, e.g. install [aiosmtp](https://pypi.org/project/aiosmtplib/) instead of using [smtplib](https://docs.python.org/3/library/smtplib.html) from the standard library. Alternatively you can wrap the sync code in a thread or process which brings in the problems explained above.

Unlike Go, the Rust programming language has nothing like go-routines built-in. Instead, there are separate runtime implementations as libraries, with [Tokio](https://tokio.rs/) being the most popular one. Tokio features a multithread-capable, async executor which is a building block for many frameworks, e.g. web servers. But as Rust also started as sync language, Tokio is also capable to bridge with sync code.  

# Conclusion

Async by design is the future of application level programming. But the sync into async code conversion on the way takes its time.
