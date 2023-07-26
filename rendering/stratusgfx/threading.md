---
permalink: /rendering/stratusgfx/threading
title: Stratus Multithreading and Async
---

![logo](/assets/v0.10/StratusLogo_Banner.jpg)

In this post I want to walk through the way that the threading and async model is designed for Stratus.

# Table of Contents
* TOC
{:toc}

# Design Goals 

These are a few of the things I was looking for when first desining these systems:

1) Engine modules and application shouldn't need to spawn their own threads for anything
2) It should be easy to queue up work on any thread
3) When a thread adds an async callback, it should be the one to execute the callback when the job finishes

Number 1 meant that all threads would be considered shared resources. Because of this it's an error for any work item to enter into an infinite loop. They need to follow the pattern of do some finite amount of work and then return.

Number 3 meant that if I had thread A creating async routines to run in parallel, when those jobs completed, thread A would be the one to execute the callbacks. This means that thread A doesn't need to deal with any synchronization - it happens behind the scenes. More on this a little later.

# Thread Setup

Two groups of threads are started everytime the engine boots up. The first is the Application/Rendering thread. This processes the main loop. The second is a set of task threads managed by the TaskSystem engine module.

Each thread, once started, is meant to run indefinitely until shutdown. They all manage their own local task list which they continuously check to see if new work has been added.

Each frame the engine and TaskSystem make sure that work added during the previous frame is committed to the task lists. This means that by default new work is scheduled in a deferred state during the lifetime of the frame it was submitted, but by the next frame it will be in the active task list.

# Thread API

{% highlight c++ %}
stratus::Thread& stratus::Thread::Current()
{% endhighlight %}

This works similarly to std::this_thread and it returns a reference to the current thread.

{% highlight c++ %}
typedef std::function<void(void)> ThreadFunction;
{% endhighlight %}

Threads accept a function object taking no arguments and returning nothing. Use capture lists to capture arguments.

{% highlight c++ %}
void stratus::QueueMany(const E& functions)
void stratus::Queue(const ThreadFunction& function)
{% endhighlight %}

This is how new work is added to a thread. For example, `stratus::Thread::Current().Queue([](){})` would queue up a new function that does nothing.

# Async API

Async objects are designed to be executed on a stratus::Thread. For most template specializations of Async, the return type is a pointer. However, for Async\<void\> there is no return.

{% highlight c++ %}
template<typename E>
class Async;

// Function computing the result will need to return bool as pointer
// Ex: new bool(true)
Async<bool> as;

// Function computing the result does not return anything
Async<bool> vs;
{% endhighlight %}

Here are the most important Async functions:

{% highlight c++ %}
// Checks if the computation failed
bool Failed() const;

// Checks if the computation function finished
// (can be true while Failed() is also true)
bool Completed() const;

// Returns either an error message or exception message,
// depending on why the function failed
std::string ExceptionMessage() const;

// Allows the caller to request that it be notified via
// callback when the async function completes
//
// This gets called even if the async function completed with
// an error! (Completed() == true, Failed() == true)
//
// Example callback: std::function<void(Async<bool>)>
typedef std::function<void(Async<E>)> AsyncCallback;
void AddCallback(const AsyncCallback & callback);

// The following are not available for Async<void>
//
// Each one provides a way of getting the result of the async 
// function. It is an error to call these before Completed() == true
// and while Failed() == true.
const E& Get() const;
E& Get();
std::shared_ptr<E> GetPtr();

{% endhighlight %}

When using `AddCallback`, the thread that calls that function will be the thread that executes the callback. This means that if you are adding all callbacks on the rendering thread, you are guaranteed that they will be executed on the rendering thread once the async functions complete.

# Task System

During startup the engine starts up a number of task threads. They do not have any specific function and instead serve as general helper threads that any engine module or the application can make use of to parallelize their work.

{% highlight c++ %}
auto * as = INSTANCE(ApplicationThread);
auto * ts = INSTANCE(TaskSystem);
{% endhighlight %}

The first line returns a pointer to the Application/Rendering thread. This is useful if you have a task that runs in parallel but at the end schedules work to execute on the main rendering thread.

The second will return a pointer to the global task system managed by the engine. Here is an example of the task system being used to schedule one async int operation:

{% highlight c++ %}
// First schedule the work
Async<int> work = ts->ScheduleTask([](){
    int numberOfItemsProcessed = 0;
    for (int i = 0; i < 10000; ++i) {
        // Perform some per index work and increment
        // items processed when needed
    }

    return new int(numberOfItemsProcessed);
});

// Now add a callback
work.AddCallback([](Async<int> result) {
    // Check if something went wrong
    if (result.Failed()) {
        std::cout << "Failed with error: " 
                  << result.ExceptionMessage() 
                  << std::endl;
        return;
    }

    std::cout << "Num items processed: "
              << result.Get()
              << std::endl;
});

{% endhighlight %}

# Many Tasks With Task System

In some cases it is necessary to submit many work items and then add a callback to see when the entire task group finishes. This can be done in the following way and works with both generic Async\<E\> as well Async\<void\>.

{% highlight c++ %}
std::vector<Async<void>> tasks;

for (int i = 0; i < 100; ++i) {
    tasks.push_back(ts->ScheduleTask([](){
        // Per task code goes here
    }));
}

// Now add a callback for the entire group
const auto callback = [](const std::vector<Async<void>>& taskList) {
    // Iterate over each async task and check if it failed
};
ts->AddGroupCallback(callback, tasks);
{% endhighlight %}

Now once every member of the async group completes (even if one or more fail with an exception), your callback will be notified.

This AddGroupCallback function follows the same rule as Async\<E\>.AddCallback meaning that the thread that adds the callback is the same thread that executes the callback.