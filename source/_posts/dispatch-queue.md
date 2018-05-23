---
title: C++ 实现的 Dispatch Queue
date: 2018-03-10 19:35:45
categories: [开发笔记]
tags: [线程]
---

iOS/Mac 平台下 apple 提供了非常好用的 `dispatch_queue` 能够很方便的进行线程的管理以及各个线程之间的切换（当然还有很多其他特性）。虽说 C++ 的标准库中提供了很多线程管理的方法，但相比于 `dispatch_queue` 还是弱爆了。由于项目中会经常用到，`GitHub` 上找了一些类似的实现都不太理想，于是自己实现了一版简单的主要支持一下特性：

1. 支持并发调用，并支持并发的任务处理
2. 支持任务的同步执行和异步执行
3. 可创建指定数量的任务线程
4. 执行同步任务时，在任务线程中可继续执行同步任务而不卡死
5. 同一任务可重复执行
6. 支持 lambda 表达式
5. 支持任务线程的安全退出

<!--more-->

### 结构设计

![dispatch_queue](/uploads/dispatch_queue_class.png)

上图是 `DispatchQueue` 类结构图，结构比较简单，`DispatchQueue` 是核心抽象类；`QueueTask` 就是任务的抽象基类了，`ClosureTask` 类是一个模板类，主要用来实现 lambda 表达式；`DisruptorImp` 与  `MutexQueueImp` 则是两个具体的 `DispatchQueue` 的实现，`disruptor` 包是一个第三方库，后面会有详细介绍，。

#### DispatchQueue 抽象类的定义

以下是 `DispatchQueue` 接口的定义和实现：

```
class DispatchQueue {
public:
    DispatchQueue(int thread_count) {}
    
    virtual ~DispatchQueue() {}
    
    template <class T, typename std::enable_if<std::is_copy_constructible<T>::value>::type* = nullptr>
    void sync(const T &task) {
        sync(std::shared_ptr<QueueTask>(new ClosureTask<T>(task)));
    }
    
    void sync(std::shared_ptr<QueueTask> task) {
        if( nullptr != task ) {
            sync_imp(task);
        }
    }
    
    template <class T, typename std::enable_if<std::is_copy_constructible<T>::value>::type* = nullptr>
    int64_t async(const T &task) {
        return async(std::shared_ptr<QueueTask>(new ClosureTask<T>(task)));
    }
    
    int64_t async(std::shared_ptr<QueueTask> task) {
        if ( nullptr != task ) {
            return async_imp(task);
        }
        return -1;
    }
    
protected:
    virtual void sync_imp(std::shared_ptr<QueueTask> task) = 0;
    
    virtual int64_t async_imp(std::shared_ptr<QueueTask> task) = 0;
};
```

`DispatchQueue` 是基于建造者模式进行设计，将同步方法、异步方法对 lambda 表达式的支持，放在父类中实现，因为这部分代码相对固定，而具体的同步与异步的实现都推迟到子类中实现，因为这部分可以有不同的实现方式。抽象的父类相对简单，只实现了对 lambda 的支持，在来看看 `QueueTask` 的类设计如下：

```
class QueueTask {
public:
    QueueTask() : _signal(false) {}

    virtual ~QueueTask() {}
    
    virtual void run() = 0;

    virtual void signal() {
        _signal = true;
        _condition.notify_all();
    }

    virtual void wait() {
        std::unique_lock <std::mutex> lock(_mutex);
        _condition.wait(lock, [this](){ return _signal; });
        _signal = false;
    }
    
    virtual void reset() {
        _signal = false;
    }

private:
    bool _signal;
    std::mutex _mutex;
    std::condition_variable _condition;
};

template <class T>
class ClosureTask : public QueueTask {
public:
    explicit ClosureTask(const T& closure) : _closure(closure) {}
private:
    void run() override {
        _closure();
    }
    T _closure;
};
```

*`virtual void run() = 0`* 是纯虚函数用来给子类重载，*`signal()/wait()`* 是两个具体方法用来支持任务的同步执行，在生产线程中执行 `wait()` 方法用来等待当前任务执行完成，而在任务线程中 `QueueTask` 执行完成后会调用 `signal` 来通知生产线程完成等待。`reset` 方法可以让该任务执行完成后重置内部状态，以便可继续将当前任务添加都队列中。`ClosureTask` 模板类则用来包装 lambda 表达式。

#### 基于 std::mutex/std::queue 的实现

基于标准库实现的思路很简单，使用标准库中提供的 `std::mutex`  和 `std::queue` ，在进行插入任务和执行任务时，对任务队列进行加锁操作，这里使用递归锁 `std::recursive_mutex` 而非 `std::mutex`。具体实现代码如下：

```
class MutexQueueImp : public DispatchQueue {
public:
    MutexQueueImp(int thread_count)
        : DispatchQueue(thread_count),
        _cancel(false),
        _thread_count(thread_count) {
        for ( int i = 0; i < thread_count; i++ ) {
            create_thread();
        }
    }

    ~MutexQueueImp() {
        _cancel = true;
        _condition.notify_all();
        for ( auto& future : _futures ) {
            future.wait();
        }
    }
    
    void sync_imp(std::shared_ptr<QueueTask> task) override {
        if ( _thread_count == 1 && _thread_id == std::this_thread::get_id() ) {
            task->reset();
            task->run();
            task->signal();
        } else {
            async_imp(task);
            task->wait();
        }
    }
    
    int64_t async_imp(std::shared_ptr<QueueTask> task) override {
        _mutex.lock();
        task->reset();
        _dispatch_tasks.push(task);
        _mutex.unlock();
        _condition.notify_one();
        return 0;
    }
    
private:
    
    MutexQueueImp(const MutexQueueImp&);
    
    void create_thread() {
        _futures.emplace_back(std::async(std::launch::async, [&]() {
            _thread_id = std::this_thread::get_id();
            while (!_cancel) {
                
                {
                    std::unique_lock <std::mutex> work_signal(_signal_mutex);
                    _condition.wait(work_signal, [this](){
                        return _cancel || !_dispatch_tasks.empty();
                    });
                }
                
                while (!_dispatch_tasks.empty() && !_cancel) {
                    _mutex.lock();
                    if ( _dispatch_tasks.empty() ) {
                        _mutex.unlock();
                        break;
                    }
                    std::shared_ptr<QueueTask> task(_dispatch_tasks.front());
                    _dispatch_tasks.pop();
                    _mutex.unlock();
                    if ( nullptr != task ) {
                        task->run();
                        task->signal();
                    }
                }
            }
        }));
    }
    
private:
    std::vector<std::future<void>> _futures;
    std::queue<std::shared_ptr<QueueTask>> _dispatch_tasks;
    std::recursive_mutex _mutex;
    bool _cancel;
    std::mutex _signal_mutex;
    std::condition_variable _condition;
    int _thread_count;
    std::thread::id _thread_id;
};
```

该类除了实现父类的`sync_imp/async_imp` 方法外，还有用来创建线程的  `create_thread` 方法，该方法每调用一次可以产生一个新的任务线程；在类的析构方法中会停止当前所有线程，并等待线程的安全退出。

在 `MutexQueueImp` 实现中，是典型的 `生产者-消费者` 线程模型，在 `async_imp` 方法中将任务插入到队列中，并通知任务线程；任务线程接收到任务加入队列的信号后，循环的从任务队列中取出任务，当所有任务处理完成，进入到休眠模式，直到下一个任务加入队列。

在 `sync_imp` 方法中会判断当前线程是否为任务线程，如果是任务线程，并且只有一个任务线程时，则直接执行任务，以免造成当前线程等待自己的情况，以免造成死锁。

#### 基于 disruptor 的实现

`Disruptor` 最初是在 Java 上被发明的，这里使用它的 C++ 实现版本原理和 Java 版本是一致的。但由于 `Disruptor` 是基于 `发布者-订阅者` 的分发模型，所以当一个任务来到时，所以等待的线程，都将被唤醒，该任务可能被多个线程同时执行，在当前我们的 `Dispatch Queue` 中是不被允许的，所以只有在单线程时，才会使用 `Disruptor` 来作为 `Dispatch Queue` 的实现，以确保高效和正确性。

> C++ 版本的阻塞等待工具类 BlockingStrategy 的实现有错误，无法唤醒，所以不要使用 BlockingStrategy 作为等待策略。

以下是 `disruptor` 实现的主要代码：

```
class DisruptorImp : public DispatchQueue {
    
public:
    DisruptorImp()
        : DispatchQueue(1),
        _cancel(false) {
        _sequencer = new DisruptorSequencer(_calls);
        create_thread();
    }
    
    ~DisruptorImp() {
        _cancel = true;
        int64_t seq = _sequencer->Claim();
        (*_sequencer)[seq] = nullptr;
        _sequencer->Publish(seq);
        _future.wait();
        delete _sequencer;
    }
    
private:
    void sync_imp(std::shared_ptr<QueueTask> task) override {
        if ( _thread_id == std::this_thread::get_id() ) {
            task->reset();
            task->run();
            task->signal();
        } else {
            async_imp(task);
            task->wait();
        }
    }
    
    int64_t async_imp(std::shared_ptr<QueueTask> task) override {
        task->reset();
        int64_t seq = _sequencer->Claim();
        (*_sequencer)[seq] = task;
        _sequencer->Publish(seq);
        return 0;
    }
    
    void create_thread() {
        _future = std::async(std::launch::async, [&]() {
            _thread_id = std::this_thread::get_id();
            this->run();
        });
    }
    
    void run() {
        int64_t seqWant(disruptor::kFirstSequenceValue);
        int64_t seqGeted, i;
        std::vector<disruptor::Sequence*> dependents;
        disruptor::SequenceBarrier<kWaitStrategy>* barrier;
        
        disruptor::Sequence handled(disruptor::kInitialCursorValue);
        dependents.push_back(&handled);
        _sequencer->set_gating_sequences(dependents);
        dependents.clear();
        barrier = _sequencer->NewBarrier(dependents);
        
        while (!_cancel)
        {
            seqGeted = barrier->WaitFor(seqWant);
            for (i = seqWant; i <= seqGeted; i++)
            {
                std::shared_ptr<QueueTask> task((*_sequencer)[i]);
                (*_sequencer)[i] = nullptr;
                handled.set_sequence(i);
                if ( nullptr != task ) {
                    task->run();
                    task->signal();
                }
            }
            seqWant = seqGeted + 1;
        }
        
        delete barrier;
    }
    
private:
    DisruptorSequencer *_sequencer;
    std::array<std::shared_ptr<QueueTask>, disruptor::kDefaultRingBufferSize> _calls;
    bool _cancel;
    std::future<void> _future;
    std::thread::id _thread_id;
};
```

`DisruptorImp` 类的结构与 `MutexQueueImp` 基本一致，只有换成了 `Disruptor` 实现，至于对 `Disruptor` 的使用可参看文章后面的链接即可。最后 `DispatchQueue` 接口对象的创建使用一下方法来创建：

```
std::shared_ptr<DispatchQueue> create(int thread_count) {
    if ( 1 == thread_count ) {
        return std::shared_ptr<DispatchQueue>(new DisruptorImp());
    } else {
        return std::shared_ptr<DispatchQueue>(new MutexQueueImp(thread_count));
    }
}
```

### 总结

`DispatchQueue` 可以说满足了基本的线程管理的需要，配合 lambda 表达式，使用起来也非常方便了。当然你也把它当做线程池来使用。后续如果有需要可以方便的扩展其他特性例如：延迟执行、任务之间的依赖关系等，完整的代码可在我的 [GitHub](https://github.com/EnkiChen/DispatchQueue) 上找到。

### 参考资料

* [并发框架 Disruptor](http://ifeve.com/disruptor/)
* [Disruptor c++ 使用指南](http://blog.csdn.net/bjrxyz/article/details/53084539)