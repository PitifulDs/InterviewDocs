# C++ 多线程高频面试题总结

## 1. 两个线程交替打印奇数和偶数

### 题目

使用两个线程，一个打印奇数，一个打印偶数，输出顺序如下：

```
1 2 3 4 5 6 7 8 ...
```

### 思路

- 使用共享变量 `num`
- 使用 `mutex` 保证互斥
- 使用 `condition_variable` 控制线程唤醒
- 奇数线程只在 `num % 2 == 1` 时执行
- 偶数线程只在 `num % 2 == 0` 时执行

### 代码

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>

using namespace std;

mutex mtx;
condition_variable cv;
int num = 1;
int n = 10;

void printOdd() {
    while (num <= n) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [] { return num % 2 == 1; });

        cout << num << " ";
        num++;

        cv.notify_all();
    }
}

void printEven() {
    while (num <= n) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [] { return num % 2 == 0; });

        cout << num << " ";
        num++;

        cv.notify_all();
    }
}

int main() {
    thread t1(printOdd);
    thread t2(printEven);

    t1.join();
    t2.join();
}
```

---

# 2. 三个线程顺序打印 ABC

### 题目

三个线程循环打印：

```
ABCABCABCABC
```

### 思路

使用 `condition_variable` + 状态变量控制执行顺序。

### 代码

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>

using namespace std;

mutex mtx;
condition_variable cv;
int state = 0;

void printA() {
    for(int i=0;i<5;i++){
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, []{ return state == 0; });

        cout<<"A";

        state = 1;
        cv.notify_all();
    }
}

void printB() {
    for(int i=0;i<5;i++){
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, []{ return state == 1; });

        cout<<"B";

        state = 2;
        cv.notify_all();
    }
}

void printC() {
    for(int i=0;i<5;i++){
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, []{ return state == 2; });

        cout<<"C";

        state = 0;
        cv.notify_all();
    }
}

int main() {
    thread t1(printA);
    thread t2(printB);
    thread t3(printC);

    t1.join();
    t2.join();
    t3.join();
}
```

---

# 3. 生产者消费者模型

### 题目

实现一个线程安全的生产者消费者模型。

### 思路

- 使用 `queue`
- 使用 `mutex` 保护共享资源
- 使用 `condition_variable` 控制生产和消费

### 代码

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <queue>
#include <condition_variable>

using namespace std;

queue<int> q;
mutex mtx;
condition_variable cv;

void producer() {
    for(int i=0;i<10;i++){
        unique_lock<mutex> lock(mtx);

        q.push(i);
        cout<<"produce "<<i<<endl;

        cv.notify_one();
    }
}

void consumer() {
    while(true){
        unique_lock<mutex> lock(mtx);

        cv.wait(lock, []{ return !q.empty(); });

        int val = q.front();
        q.pop();

        cout<<"consume "<<val<<endl;
    }
}

int main() {
    thread p(producer);
    thread c(consumer);

    p.join();
    c.join();
}
```

---

# 4. 实现线程安全队列

### 思路

- 使用 `mutex` 保证线程安全
- 使用 `condition_variable` 控制等待

### 代码

```cpp
template<typename T>
class SafeQueue {
private:
    queue<T> q;
    mutex mtx;
    condition_variable cv;

public:
    void push(T value){
        unique_lock<mutex> lock(mtx);
        q.push(value);
        cv.notify_one();
    }

    T pop(){
        unique_lock<mutex> lock(mtx);

        cv.wait(lock,[this]{ return !q.empty(); });

        T val = q.front();
        q.pop();

        return val;
    }
};
```

---

# 5. C++ 常见线程同步工具

| 工具 | 作用 |
| --- | --- |
| mutex | 互斥锁 |
| lock_guard | RAII自动释放锁 |
| unique_lock | 更灵活的锁 |
| condition_variable | 线程同步 |
| atomic | 原子操作 |
| future/promise | 线程间结果传递 |

---

# 6. 常见面试问题

### 1 什么是线程安全？

多个线程访问共享资源时不会产生数据竞争。

### 2 什么是死锁？

多个线程互相等待对方释放资源。

### 3 如何避免死锁？

- 统一加锁顺序
- 使用 `std::lock`
- 减少锁粒度

### 4 condition_variable 为什么要用 while？

防止 **虚假唤醒（spurious wakeup）**。

### 5 notify_one vs notify_all

| 函数 | 作用 |
| --- | --- |
| notify_one | 唤醒一个线程 |
| notify_all | 唤醒所有线程 |

---

# 7. C++11 线程 API

创建线程：

```cpp
thread t(func);
```

等待线程：

```cpp
t.join();
```

分离线程：

```cpp
t.detach();
```

---

# 8. 高频面试线程题

1. 两线程交替打印奇偶数
2. 三线程打印 ABC
3. 生产者消费者模型
4. 实现线程安全队列
5. 实现读写锁
6. 实现线程池

---

# 9. 推荐扩展

建议掌握：

- 线程池实现
- lock-free 编程
- atomic 原子操作
- 内存模型
- false sharing
- CAS