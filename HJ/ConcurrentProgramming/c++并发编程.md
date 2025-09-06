# 第4 章 

## 条件变量

```
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <vector>
#include <chrono>

class ProducerConsumer {
private:
    std::queue<int> data_queue;
    mutable std::mutex mtx; // mutable 允许在 const 函数中锁定
    std::condition_variable cv;
    bool finished = false; // 指示生产者是否已完成
    static constexpr int MAX_ITEMS = 10; // 生产者生产总数

public:
    // 生产者线程函数
    void producer() {
        for (int i = 1; i <= MAX_ITEMS; ++i) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟生产耗时
            std::unique_lock<std::mutex> lock(mtx);
            data_queue.push(i);
            std::cout << "Produced: " << i << std::endl;
            lock.unlock(); // 可以提前解锁，避免 notify 时持有锁太久
            cv.notify_one(); // 通知一个消费者
        }

        // 生产完成
        {
            std::lock_guard<std::mutex> lock(mtx);
            finished = true;
        }
        cv.notify_all(); // 通知所有消费者，生产结束了
    }

    // 消费者线程函数
    void consumer(int id) {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);

            // 关键：使用 wait(lock, predicate) 处理虚假唤醒和条件检查
            cv.wait(lock, [this] { 
                return !data_queue.empty() || finished; 
            });

            // wait 返回时，lock 已被重新获取，且 predicate 为 true

            if (finished && data_queue.empty()) {
                std::cout << "Consumer " << id << " exiting (no more data).\n";
                break; // 生产结束且队列为空，退出
            }

            // 此时队列非空，可以安全消费
            int value = data_queue.front();
            data_queue.pop();
            lock.unlock(); // 提前解锁，避免 sleep 时持有锁

            std::cout << "Consumer " << id << " consumed: " << value << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(50)); // 模拟消费耗时
        }
    }
};

int main() {
    ProducerConsumer pc;

    std::vector<std::thread> consumers;
    const int NUM_CONSUMERS = 3;

    // 启动多个消费者线程
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        consumers.emplace_back(&ProducerConsumer::consumer, &pc, i + 1);
    }

    // 启动生产者线程
    std::thread producer_thread(&ProducerConsumer::producer, &pc);

    // 等待所有线程完成
    producer_thread.join();
    for (auto& t : consumers) {
        t.join();
    }

    std::cout << "All done.\n";
    return 0;
}
```

## 期望

```
std::future + std::async


#include <iostream>
#include <future>
#include <thread>
#include <chrono>

// 模拟一个耗时计算
int expensive_calculation(int x) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return x * x + 1;
}

int main() {
    std::cout << "Starting async task...\n";

    // 启动异步任务，返回一个 future<int>
    // std::launch::async 表示强制在新线程运行（默认策略可能延迟执行）
    std::future<int> result_future = std::async(std::launch::async, expensive_calculation, 5);

    // 主线程可以做其他事情
    std::cout << "Doing other work while calculation is running...\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    // 尝试检查任务状态（非阻塞）
    std::future_status status = result_future.wait_for(std::chrono::milliseconds(0));
    if (status == std::future_status::ready) {
        std::cout << "Result is ready!\n";
    } else {
        std::cout << "Result is not ready yet, waiting...\n";
    }

    // 获取结果（阻塞，直到结果可用）
    // 注意：get() 只能调用一次！之后 future 为空
    int result = result_future.get(); 
    std::cout << "The result is: " << result << std::endl;

    // result_future.get(); // 错误！future 已经被 move 或 reset，再次调用 get() 会抛出异常

    return 0;
}
```

```
std::future + std::promise + std::thread

#include <iostream>
#include <future>
#include <thread>
#include <chrono>

void worker(std::promise<std::string>&& prms) {
    try {
        std::this_thread::sleep_for(std::chrono::seconds(3));
        // 模拟工作成功完成
        std::string data = "Work completed successfully!";
        prms.set_value(data); // 设置 future 的值
        // prms.set_value_at_thread_exit(data); // 可选：在线程退出时设置值
    } catch (...) {
        // 如果工作过程中抛出异常
        prms.set_exception(std::current_exception()); // 将异常设置到 future
    }
}

int main() {
    std::promise<std::string> promise;
    std::future<std::string> future = promise.get_future(); // 获取与 promise 关联的 future

    std::thread worker_thread(worker, std::move(promise)); // 启动工作线程

    std::cout << "Waiting for worker thread...\n";

    try {
        // 等待并获取结果
        std::string result = future.get(); // 如果 worker 调用了 set_exception，这里会 rethrow 异常
        std::cout << "Worker result: " << result << std::endl;
    } catch (const std::exception& e) {
        std::cout << "Worker threw exception: " << e.what() << std::endl;
    }

    worker_thread.join();
    return 0;
}
```

```


#include <iostream>
#include <future>
#include <thread>
#include <vector>

int main() {
    // 启动一个异步任务
    std::future<int> future = std::async(std::launch::async, []() {
        std::this_thread::sleep_for(std::chrono::seconds(2));
        return 42;
    });

    // --- std::future (单一所有者，结果只能取一次) ---
    // std::future<int> future2 = future; // 错误！不能复制
    std::future<int> future_moved = std::move(future); // 可以移动

    int result1 = future_moved.get(); // 第一次获取结果
    std::cout << "future result: " << result1 << std::endl;

    // int result2 = future_moved.get(); // 错误！get() 只能调用一次，future 已失效

    // --- std::shared_future (共享所有者，结果可多次获取) ---
    // 重新启动一个任务来演示 shared_future
    std::future<int> another_future = std::async(std::launch::async, []() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        return 100;
    });

    // 将 future 转换为 shared_future (通过移动构造)
    std::shared_future<int> shared_fut(std::move(another_future));

    // shared_future 可以被复制，多个线程/对象可以持有
    std::shared_future<int> shared_fut_copy1 = shared_fut;
    std::shared_future<int> shared_fut_copy2 = shared_fut;

    // 所有副本都可以安全地、非破坏性地获取结果
    std::vector<std::thread> threads;
    for (int i = 0; i < 3; ++i) {
        threads.emplace_back([shared_fut_copy = shared_fut]() { // 捕获副本
            std::this_thread::sleep_for(std::chrono::milliseconds(100 * (i+1)));
            int val = shared_fut_copy.get(); // 多次调用 get() 是安全的
            std::cout << "Thread " << i << " got value: " << val << std::endl;
        });
    }

    // 原始的 shared_fut 也可以获取
    int original_val = shared_fut.get();
    std::cout << "Main thread got value: " << original_val << std::endl;

    for (auto& t : threads) t.join();

    return 0;
}
```

