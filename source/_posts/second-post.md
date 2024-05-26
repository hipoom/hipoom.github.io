---
title: 线程安全中的 Signal Before Wait 问题
date: 2024-05-26 17:32:20
tags:
---
# 线程安全中的 Signal Before Wait 问题

标签（空格分隔）： 博客草稿

---

在多线程中使用 notify-wait 时，如果 等待线程A 在调用 wait() 之前，唤醒线程B 已经调用了 notify() 方法，会导致 等待线程A 永远得不到唤醒，一直等待下去，这就是 Signal Before Wait 问题：
![最简单的SBW问题.png-8.8kB][1]
例如下面的代码：
```C++
int main() {
    // 定义互斥量和条件变量
    std::mutex mutex;
    std::condition_variable condition;
    
    // 唤醒线程
    auto signalThread = new std::thread([&]() {
        log("即将唤醒");
        condition.notify_all();
        log("唤醒完毕");
    });
    signalThread->detach();
    
    // sleep 1秒，模拟耗时操作
    sleep(1);
    
    // 等待线程
    auto waitThread = new std::thread([&]() {
        std::unique_lock lock(mutex);
        log("即将等待");
        condition.wait(lock);
        log("等待完毕");
    });
    waitThread->detach();
    
    // 主线程5分钟后释放
    sleep(60 * 5);
    delete signalThread;
    delete waitThread;
    return 0;
}
```


为了解决这个问题，我们引入标记位 isReady 来解决( 第7、12、26行 )：
```C++
int main() {
    // 定义互斥量和条件变量
    std::mutex mutex;
    std::condition_variable condition;
    
    // 新增标记位，用于标记是否已经准备好了
    bool isReady = false;
    
    // 唤醒线程
    auto signalThread = new std::thread([&]() {
        log("即将唤醒");
        isReady = true;
        condition.notify_all();
        log("唤醒完毕");
    });
    signalThread->detach();
    
    // sleep 1秒，模拟耗时操作
    sleep(1);
    
    // 等待线程
    auto waitThread = new std::thread([&]() {
        std::unique_lock lock(mutex);
        log("即将等待");
        // 只有在没有准备好时才需要等待
        while (!isReady) {
            condition.wait(lock);
        }
        log("等待完毕");
    });
    waitThread->detach();
    
    // 主线程5分钟后释放
    sleep(60 * 5);
    delete signalThread;
    delete waitThread;
    return 0;
}
```
实际上，只添加这个标记位，是没有解决问题的，如果在 while -> wait 之间，isReady 被修改为 true，依然会导致等待线程陷入等待状态：
![while-wait之间对标记位做了赋值.png-22kB][2]

因此，我们需要把 while-wait 和 isReady更新 都要放到临界区中。
由于 unique_lock 在构造方法中自动调用了 mutex.lock() 方法，且 condition.wait() 在移到等待队列前会自动调用 mutex.unlock()，所以 等待线程A 的 while-wait 本来就在临界区中了，只需修改唤醒线程的代码：
```C++
// 唤醒线程
auto signalThread = new std::thread([&]() {
    log("即将唤醒");
    mutex.lock();
    isReady = true;
    condition.notify_all();
    mutex.unlock();
    log("唤醒完毕");
});
signalThread->detach();
```
这里有个细节是 notify() 要不要放在临界区中。
如果放到临界区中，可能存在多余的线程上下文切换：


如果 notify() 在 unlock() 之前调用，那么 等待线程A 在收到唤醒时，会尝试 lock()，但 唤醒线程B 还没有 unlock()，所以 等待线程A 会再次进入休眠。这就多了2次线程的上下文切换。
那如果 notify() 在 unlock() 之后会不会有问题呢？当然也可能有问题的，例如有三个线程时：
这会导致虚假唤醒。如果业务更复杂，可能会导致意料之外的其他错误。
如果对性能要求非常严格，且 unlock - notify 之间不会有意料之外的逻辑，那可以把 unlock 放到 notify 之前；
如果不在意这种细微的性能损失，就把 notify 放到 unlock 之前。这样能避免很多隐秘的BUG。
 
---- 
 
源码：

```
void log(const std::string& msg) {
    std::cout << msg << std::endl;
}
int main() {
    // 定义互斥量和条件变量
    std::mutex mutex;
    std::condition_variable condition;
    // 新增标记位，用于标记是否已经准备好了
    bool isReady = false;
    // 唤醒线程
    auto signalThread = new std::thread([&]() {
        log("即将唤醒");
        isReady = true;
        condition.notify_all();
        log("唤醒完毕");
    });
    signalThread->detach();
    // sleep 1秒，模拟耗时操作
    sleep(1);
    // 等待线程
    auto waitThread = new std::thread([&]() {
        std::unique_lock lock(mutex);
        log("即将等待");
        // 只有在没有准备好时才需要等待
        while (!isReady) {
            condition.wait(lock);
        }
        log("等待完毕");
    });
    waitThread->detach();
    // 主线程5分钟后释放
    sleep(60 * 5);
    delete signalThread;
    delete waitThread;
    return 0;
}
void core1() {
    std::mutex mutex;
    std::condition_variable condition;
    bool isReady = false;
    auto signalThread = new std::thread([&condition, &isReady, &mutex]() {
        // ....
        sleep(2);
        mutex.lock();
        isReady = true;
        condition.notify_all(); // ①
        mutex.unlock();
    });
    signalThread->detach();
    // while   ---> notify   ---> wait
    auto anotherWaitThread1 = new std::thread([&condition, &isReady, &mutex]() {
        // unique_lock 的构造函数会调用传入 mutex 的 lock 函数。
        std::unique_lock lock(mutex);
        while(!isReady) {
            // condition 的 wait 函数会释放锁，并且会阻塞当前线程；当被唤醒后，会再次抢占锁。
            condition.wait(lock);
        }
    });
    anotherWaitThread1->detach();
    sleep(1000);
}
void core() {
    std::mutex mutex;
    std::condition_variable condition;
    bool isReady = false;
    auto subThread = new std::thread([&condition, &isReady, &mutex]() {
        sleep(1);
        std::cout << "update" << std::endl;
        isReady = true;         // ②
        sleep(1);
        mutex.lock();
        std::cout << "notify..." << std::endl;
        condition.notify_all(); // ①
        mutex.unlock();
        // 关于 ① ② 的顺序：
        // 如果先 notify，再更新标记位，可能出现：  等待方进入了while  ->  notify  -> update flag -> wait 的情况。
        // 如果先更新标记位，再 notify，可能出现：  等待方进入了while  ->  update flag -> notify  -> wait 的情况。
        // 都依然会导致 signal before wait。
        // 所以，while-wait 和 update-notify  都需要处于临界区中。
        // 准确地说，  while 和 wait 之间，不能有 notify。
        // while -> wait -> notify -> update  没问题
        // while -> wait -> update -> notify  没问题
        // while -> notify -> wait -> update  有问题
        // while -> update -> wait -> notify  没问题
        // while -> notify -> update -> wait  有问题
        // while -> update -> notify -> wait  有问题
    });
    subThread->detach();
    std::unique_lock lock(mutex);
    mutex.lock();
    std::cout << "while..." << std::endl;
    while (true) {
        if (isReady) {
            break;
        }
        std::cout << "in while" << std::endl;
        sleep(2);
        std::cout << "wait" << std::endl;
        condition.wait(lock);
    }
    mutex.unlock();
    std::cout << "...while" << std::endl;
}
int main2() {
    core1();
    return 0;
}
```

  [1]: http://static.zybuluo.com/SinoSnack/6db24wehl9dxamm9mp1kpak2/%E6%9C%80%E7%AE%80%E5%8D%95%E7%9A%84SBW%E9%97%AE%E9%A2%98.png
  [2]: http://static.zybuluo.com/SinoSnack/9rdda3qs10ht7nmgmygv7ji9/while-wait%E4%B9%8B%E9%97%B4%E5%AF%B9%E6%A0%87%E8%AE%B0%E4%BD%8D%E5%81%9A%E4%BA%86%E8%B5%8B%E5%80%BC.png