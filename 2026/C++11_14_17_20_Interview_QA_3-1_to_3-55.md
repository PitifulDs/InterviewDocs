# C++11/14/17/20 高频面试题深度答案（3-1 ~ 3-55）

> 主题覆盖：智能指针、移动语义、完美转发、并发同步、lambda、auto/decltype、shared_ptr 底层、C++20 协程、现代 C++ 函数式编程。  
> 写法目标：尽量贴近面试可复述风格，同时补足底层原理、使用场景、代码示例与常见坑点。

---

## 3-1 C++ 智能指针，使用场景

### 核心答案
C++ 智能指针是基于 RAII 的资源管理工具，用来自动释放动态资源，避免内存泄漏、重复释放和异常路径下的资源丢失。最常用的有 `unique_ptr`、`shared_ptr`、`weak_ptr`。

### 三类智能指针

#### 1）`std::unique_ptr`
- 独占所有权
- 不能拷贝，只能移动
- 开销最小，默认首选

**适用场景：**
- 明确只有一个对象负责释放资源
- 类成员独占资源
- 工厂函数返回对象
- 树形结构中父节点独占子节点

```cpp
std::unique_ptr<Foo> p = std::make_unique<Foo>();
```

#### 2）`std::shared_ptr`
- 共享所有权
- 底层有引用计数
- 最后一个拥有者销毁时释放资源

**适用场景：**
- 多个模块共享同一对象生命周期
- 异步任务、回调中需要延长对象生命周期
- 图结构、资源管理器中多个实体共同持有对象

```cpp
auto sp = std::make_shared<Foo>();
```

#### 3）`std::weak_ptr`
- 弱引用，不拥有对象
- 不增加强引用计数
- 一般配合 `shared_ptr` 使用

**适用场景：**
- 打破循环引用
- 缓存
- 观察者模式
- 异步回调中避免悬空 `this`

### 面试补充
默认优先 `unique_ptr`，只有“确实需要共享生命周期”时才使用 `shared_ptr`。`weak_ptr` 主要解决“只想引用，不想拥有”的问题。

---

## 3-2 智能指针？环形引用？怎么解决？

### 核心答案
环形引用通常发生在两个对象互相持有 `shared_ptr`，导致强引用计数永远不为 0，对象无法析构，形成内存泄漏。

### 示例
```cpp
struct B;
struct A {
    std::shared_ptr<B> b;
};
struct B {
    std::shared_ptr<A> a;
};
```

如果 `A` 和 `B` 互相持有 `shared_ptr`，当外部指针都释放后：
- `A` 内部还有一个指向 `B` 的强引用
- `B` 内部还有一个指向 `A` 的强引用
- 两边计数都不为 0，无法释放

### 解决方法
一端改成 `weak_ptr`：

```cpp
struct B;
struct A {
    std::shared_ptr<B> b;
};
struct B {
    std::weak_ptr<A> a;
};
```

### 为什么有效
因为 `weak_ptr` 不增加强引用计数。对象销毁条件只看强引用计数，`weak_ptr` 只是观察者。

### 面试追问
**问：是不是所有双向关系都要用 `weak_ptr`？**  
不是。关键看“拥有关系”。通常一端拥有，一端观察。比如父节点拥有子节点，子节点只弱引用父节点。

---

## 3-3 什么时候用智能指针？什么时候用裸指针？

### 核心答案
看“是否拥有资源”。
- **拥有资源**：用智能指针
- **不拥有，只借用/观察**：用裸指针或引用

### 适合用智能指针的场景
1. 需要自动释放堆对象
2. 存在异常或多返回路径
3. 生命周期复杂，容易忘记 `delete`
4. 需要明确表达所有权语义

### 适合用裸指针或引用的场景
1. 函数参数只是借用对象
2. 明确不负责释放资源
3. 性能敏感且生命周期非常清晰
4. 仅表示“可空观察者”时可以用裸指针

```cpp
void use(Foo* p);        // 可空借用，不拥有
void use(Foo& p);        // 不可空借用，不拥有
void take(std::unique_ptr<Foo> p); // 接管所有权
```

### 面试建议回答
“我一般按所有权来区分。拥有对象时优先用 `unique_ptr`；需要共享生命周期时才用 `shared_ptr`；函数参数如果只是访问对象，不转移所有权，则优先用引用或裸指针表达借用语义。”

---

## 3-4 weak_ptr 了解吗？

### 核心答案
`weak_ptr` 是一种弱引用智能指针，它不拥有对象，也不会增加 `shared_ptr` 的强引用计数。主要作用是：
1. 解决 `shared_ptr` 循环引用
2. 在不延长对象生命周期的前提下安全观察对象

### 常用接口
- `expired()`：对象是否已销毁
- `lock()`：尝试拿到一个 `shared_ptr`

```cpp
std::weak_ptr<Foo> wp = sp;
if (auto p = wp.lock()) {
    p->work();
}
```

### 面试追问
**问：`weak_ptr` 能直接解引用吗？**  
不能。必须先 `lock()`，拿到有效的 `shared_ptr` 才能访问对象。

---

## 3-5 我们一般不会发生循环引用，weak_ptr 还有什么使用场景？

### 核心答案
除了打破循环引用，`weak_ptr` 还有很多工程用途。

### 典型场景

#### 1）缓存
缓存不应该强行延长对象生命周期。可以把缓存表里的值存成 `weak_ptr`，需要时 `lock()`：

```cpp
std::unordered_map<int, std::weak_ptr<Foo>> cache;
```

#### 2）观察者模式
被观察对象不应该因为订阅者存在就永远不释放。Subject 可以存订阅者的 `weak_ptr`。

#### 3）异步回调 / 定时器
回调执行时对象可能已经析构，这时不要直接捕获 `this`，而是捕获 `weak_ptr`：

```cpp
std::weak_ptr<Session> wk = shared_from_this();
executor.submit([wk] {
    if (auto self = wk.lock()) {
        self->doWork();
    }
});
```

#### 4）GUI / 节点图 / 场景图
父节点强拥有子节点，子节点弱引用父节点。

### 结论
`weak_ptr` 的本质用途不是“专门解决循环引用”，而是“表达一种非拥有关系”。

---

## 3-6 shared_ptr 能进行拷贝构造吗

### 核心答案
可以。`shared_ptr` 的设计目标就是共享所有权，拷贝构造会让多个 `shared_ptr` 指向同一对象，并让强引用计数加一。

```cpp
auto p1 = std::make_shared<int>(10);
std::shared_ptr<int> p2 = p1; // 拷贝构造，引用计数 +1
```

### 原理
`shared_ptr` 内部通常保存：
- 对象指针
- 控制块指针

拷贝构造时，不是复制对象，而是：
- 复制对象指针
- 复制控制块指针
- 控制块中的强引用计数加一

### 面试追问
**问：`unique_ptr` 为什么不能拷贝，`shared_ptr` 却可以？**  
因为 `unique_ptr` 表达独占所有权，拷贝会导致两个拥有者；`shared_ptr` 表达共享所有权，拷贝正是其设计目标。

---

## 3-7 你对 auto 的理解

### 核心答案
`auto` 是 C++11 引入的类型推导关键字，用于让编译器在编译期根据初始化表达式推导变量类型。本质上它的推导规则和模板类型推导非常接近。

### 作用
1. 减少冗长类型书写
2. 避免模板、迭代器类型难写
3. 提高代码可读性和可维护性
4. 配合泛型编程更自然

### 示例
```cpp
std::vector<std::pair<int, std::string>> v;
auto it = v.begin();
```

### 重要规则
- `auto` 会忽略顶层 `const`
- `auto` 默认不会保留引用

```cpp
int x = 10;
int& rx = x;
auto a = rx;   // a 是 int，不是 int&
auto& b = rx;  // b 是 int&
```

### 面试追问
**问：`auto` 会不会隐藏问题？**  
会。比如不小心拷贝了对象、丢了引用或 `const`。所以在范围 `for` 里，修改元素通常要用 `auto&`。

---

## 3-8 原子操作？

### 核心答案
原子操作是不可被线程调度打断的最小操作单位。在 C++ 中通常用 `std::atomic<T>` 来实现，用于保证多线程下对单个共享变量的读写不会产生数据竞争。

### 为什么需要原子操作
多个线程同时修改一个普通变量会产生未定义行为：

```cpp
int cnt = 0;
// 多线程 ++cnt 是有数据竞争的
```

改成：

```cpp
std::atomic<int> cnt{0};
cnt.fetch_add(1);
```

### 常见原子操作
- `load()`
- `store()`
- `fetch_add()`
- `compare_exchange_weak/strong()`

### 底层原理
通常依赖 CPU 提供的原子指令，比如 CAS（Compare-And-Swap）或总线锁 / cache lock。

### 原子 vs 锁
- 原子适合单变量、简单状态
- 锁适合多个变量的一致性保护和复杂临界区

### 面试追问
**问：原子就一定比锁快吗？**  
不一定。高冲突时原子可能自旋很严重；复杂逻辑下用锁反而更简单、更稳。

---

## 3-9 move 语义？move 的实现、意义、应用场景

### 核心答案
移动语义的本质是：**把资源所有权从一个对象转移到另一个对象，而不是深拷贝资源**。它的目标是减少不必要的资源复制，提高性能。

### 为什么需要移动语义
对于拥有堆内存、文件句柄、socket 的对象，深拷贝代价大：
- 重新申请资源
- 复制内容
- 异常处理复杂

如果对象是临时对象或即将被销毁，就没必要深拷贝，直接“偷资源”即可。

### 示例：移动构造
```cpp
class Buffer {
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {}

    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    ~Buffer() { delete[] data_; }

private:
    int* data_{nullptr};
    size_t size_{0};
};
```

### 意义
1. 提高性能
2. 支持容器高效扩容与插入
3. 避免大对象深拷贝
4. 让资源类更高效地转移所有权

### 应用场景
- `std::vector` 扩容搬迁元素
- 函数返回大对象
- `unique_ptr` 所有权转移
- 线程池任务封装

### 面试追问
**问：移动后的对象还能用吗？**  
能析构、能赋值，但内部状态通常是“有效但未指定”。不要依赖它原来的值。

---

## 3-10 std::move 了解吗？底层实现什么样的

### 核心答案
`std::move` 本身不移动任何资源，它只是一个类型转换工具，把一个左值强制转换成右值引用，从而触发移动构造或移动赋值。

### 典型实现
```cpp
template <class T>
typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}
```

### 关键点
- `std::move(x)` 后，`x` 仍然是对象
- 只是把 `x` 标记为“可以被当成右值使用”
- 真正发生移动的是后续调用的移动构造 / 移动赋值

### 面试追问
**问：`std::move` 一定会触发移动吗？**  
不一定。如果类型没有移动构造，可能退化成拷贝；如果目标接口不支持移动，也不会真正移动。

---

## 3-11 左值（lvalue）与右值（rvalue）

### 核心答案
- **左值**：有身份、通常可取地址、生命周期较明确的对象
- **右值**：临时值、将亡值、没有稳定身份的表达式结果

### 示例
```cpp
int x = 10;   // x 是左值
int y = x + 1; // x + 1 是右值
```

### 更准确的理解
现代 C++ 中还细分为：
- lvalue
- xvalue（将亡值）
- prvalue（纯右值）

通常面试里把“左值 / 右值”讲清楚就够了。

### 常见判断
- 能否取地址不是唯一标准，但通常左值可取地址
- 有名字的变量通常是左值
- 临时对象通常是右值

---

## 3-12 左值引用与右值引用

### 核心答案
- 左值引用：`T&`，只能绑定左值
- 右值引用：`T&&`，只能绑定右值
- `const T&` 比较特殊，可以绑定左值和右值

### 示例
```cpp
int x = 10;
int& lref = x;   // OK
int&& rref = 20; // OK
// int&& r2 = x; // 错，x 是左值
```

### 为什么要有右值引用
为了支持移动语义和完美转发。

### 面试追问
**问：有名字的右值引用变量本身是什么？**  
是左值。比如：

```cpp
int&& rr = 10;
foo(rr); // rr 是左值
```

这是完美转发里非常关键的点。

---

## 3-13 理解 C++23 中的 std::inout_ptr 和 std::out_ptr

### 核心答案
`std::out_ptr` 和 `std::inout_ptr` 是 C++23 为了解决“智能指针与 C 风格输出参数 API 互操作”引入的适配器。

很多 C API 习惯这样写：
```cpp
void create_resource(Resource** out);
```

但 C++ 里我们常用智能指针管理资源。`out_ptr` / `inout_ptr` 就是让智能指针能方便地传给这种 `T**` 风格接口。

### 区别
- `out_ptr`：适合纯输出参数，调用前通常会重置目标
- `inout_ptr`：适合输入输出参数，调用前保留原指针，调用后更新

### 典型作用
把：
```cpp
std::unique_ptr<Resource, Deleter> p;
c_api_create(&raw);
p.reset(raw);
```
变成更标准化的方式。

### 面试表达
“它们主要是为了解决现代 C++ 智能指针和传统 `T**` 风格 C API 的桥接问题，避免手工 `release/reset` 的模板化样板代码和异常安全隐患。”

---

## 3-14 unique_ptr 怎么赋值给另一个 unique_ptr

### 核心答案
`unique_ptr` 不能拷贝，只能移动。

```cpp
std::unique_ptr<Foo> p1 = std::make_unique<Foo>();
std::unique_ptr<Foo> p2 = std::move(p1);
```

### 为什么
因为 `unique_ptr` 表达独占所有权，如果允许拷贝，就会出现两个指针同时认为自己拥有同一资源，导致重复释放。

### 移动后状态
- `p2` 获得资源
- `p1` 变为空

### 赋值运算
```cpp
p2 = std::move(p1);
```

### 面试追问
**问：能不能把 `unique_ptr` 作为函数参数传递所有权？**  
可以，通常按值传参，然后调用端 `std::move` 进去。

---

## 3-15 C++ 怎么保证线程安全

### 核心答案
C++ 保证线程安全主要靠以下几类机制：
1. 互斥锁
2. 原子操作
3. 条件变量
4. 线程局部存储
5. 不共享 / 消息传递设计
6. 智能指针 + 生命周期管理

### 常见方式

#### 1）加锁
- `std::mutex`
- `std::recursive_mutex`
- `std::timed_mutex`
- `std::shared_mutex`

#### 2）原子变量
适合简单状态和计数器：
```cpp
std::atomic<bool> running{true};
```

#### 3）条件变量
线程间同步等待某个条件成立。

#### 4）线程局部存储 `thread_local`
每个线程一份副本，天然避免竞争。

#### 5）设计层面避免共享
比如 actor 模型、消息队列、单线程拥有数据。

### 面试补充
“线程安全不是只会加锁，更重要的是明确共享数据边界，减少共享，必要时用原子、锁、条件变量配合。”

---

## 3-16 C++ 里有哪些锁？unique_lock 和 lock_guard 的区别

### 常见锁
- `std::mutex`
- `std::recursive_mutex`
- `std::timed_mutex`
- `std::recursive_timed_mutex`
- `std::shared_mutex`
- `std::shared_timed_mutex`

### `lock_guard` 和 `unique_lock` 区别

#### `lock_guard`
- 最轻量
- 构造即加锁，析构即解锁
- 不能手动解锁
- 不能延迟加锁

```cpp
std::lock_guard<std::mutex> lk(m);
```

#### `unique_lock`
- 更灵活
- 支持延迟加锁
- 支持手动 `lock()` / `unlock()`
- 支持和 `condition_variable` 配合

```cpp
std::unique_lock<std::mutex> lk(m, std::defer_lock);
lk.lock();
```

### 面试结论
- 简单作用域加锁：`lock_guard`
- 需要灵活控制加解锁：`unique_lock`
- 需要条件变量：`unique_lock`

---

## 3-17 设计一个类似 unique_lock 的锁，但创建时不加锁，想加锁时再加锁

### 核心思路
这本质就是“延迟加锁”的 RAII 封装。`std::unique_lock` 已经支持 `std::defer_lock`。

### 示例实现
```cpp
template<typename Mutex>
class MyUniqueLock {
public:
    explicit MyUniqueLock(Mutex& m) : mtx_(&m), owns_(false) {}

    void lock() {
        if (!owns_) {
            mtx_->lock();
            owns_ = true;
        }
    }

    void unlock() {
        if (owns_) {
            mtx_->unlock();
            owns_ = false;
        }
    }

    ~MyUniqueLock() {
        if (owns_) mtx_->unlock();
    }

private:
    Mutex* mtx_;
    bool owns_;
};
```

### 关键点
- 保存互斥量指针
- 保存当前是否持有锁
- 析构时如果持有则自动解锁

### 面试追问
可以补充：
- 不可拷贝
- 可以支持移动语义
- 可以增加 `owns_lock()`、`release()` 等接口

---

## 3-18 早期的智能指针 auto_ptr 为什么被废弃

### 核心答案
`auto_ptr` 的问题是：拷贝行为不是“共享”也不是“禁止”，而是**偷偷转移所有权**，这会导致非常反直觉和危险的语义，因此在 C++11 被废弃，C++17 被移除。

### 例子
```cpp
std::auto_ptr<Foo> p1(new Foo);
std::auto_ptr<Foo> p2 = p1;
```

拷贝后：
- `p2` 拿走资源
- `p1` 变成空

这会导致：
1. 拷贝语义非常奇怪
2. 容器中行为异常
3. 很容易出现悬空或空指针访问

### 为什么 `unique_ptr` 更好
`unique_ptr` 直接禁止拷贝，只允许显式移动，语义更清晰、更安全。

---

## 3-19 std::function 和 std::bind 的跨类回调问题

### 核心答案
`std::function` 是可调用对象的通用封装器，`std::bind` 用来绑定函数和参数。跨类回调时，通常用于：
- 把成员函数包装成回调
- 绑定对象实例和成员函数

### 示例
```cpp
class A {
public:
    void onEvent(int x) {
        std::cout << x << std::endl;
    }
};

A a;
std::function<void(int)> cb = std::bind(&A::onEvent, &a, std::placeholders::_1);
cb(42);
```

### 更推荐 lambda
现代 C++ 里通常更推荐 lambda：

```cpp
std::function<void(int)> cb = [&a](int x) { a.onEvent(x); };
```

### 跨类回调的风险
如果回调保存了对象地址，而对象先析构，后续回调调用会出问题。

### 安全做法
- 用 `weak_ptr` / `shared_ptr` 管理生命周期
- 避免裸 `this` 长期存活在异步回调中

---

## 3-20 匿名函数相比普通函数的区别，lambda 捕获方式有哪些，this 可以捕获吗

### 匿名函数和普通函数的区别
#### lambda 的优势
1. 可以直接写在使用位置
2. 可以捕获上下文变量
3. 常用于 STL 算法、回调、局部策略

#### 普通函数的特点
1. 没有捕获能力
2. 可复用性更强
3. 更适合独立逻辑

### lambda 捕获方式
- `[]`：不捕获
- `[=]`：值捕获全部
- `[&]`：引用捕获全部
- `[x]`：值捕获指定变量
- `[&x]`：引用捕获指定变量
- `[this]`：捕获当前对象指针
- `[*this]`（C++17）：捕获当前对象副本

### `this` 可以捕获吗
可以，`[this]` 表示捕获当前对象指针。

### 风险
`[this]` 捕获的是指针，不延长对象生命周期。如果 lambda 异步执行，而对象先析构，就会悬空。

### 更安全做法
```cpp
auto self = shared_from_this();
auto cb = [self] { self->work(); };
```
或：
```cpp
std::weak_ptr<Session> wk = shared_from_this();
auto cb = [wk] {
    if (auto self = wk.lock()) self->work();
};
```

---

## 3-21 c++11 typedef new

### 核心答案
这里通常指的是 **`using` 作为 `typedef` 的现代替代写法**。C++11 新增了 `using` 类型别名，它比 `typedef` 更直观，尤其在模板别名中优势明显。

### 对比
```cpp
typedef std::vector<int> IntVec;
using IntVec = std::vector<int>;
```

### `using` 的优势
1. 语法从左到右更自然
2. 支持模板别名

```cpp
template<typename T>
using Vec = std::vector<T>;
```

而 `typedef` 不能直接这样写模板别名。

---

## 3-22 auto 和 decltype 关键字的作用

### auto
- 编译期根据初始化表达式推导类型
- 常用于变量声明、返回值简化

### decltype
- 推导表达式的类型
- 不会执行表达式
- 更适合保留精确类型特征

### 示例
```cpp
int x = 0;
int& rx = x;
auto a = rx;       // int
decltype(rx) b = x; // int&
```

### 二者关系
- `auto` 更偏“根据初始化值推导变量类型”
- `decltype` 更偏“获得某个表达式的精确类型”

### 面试补充
`decltype(auto)` 常用于保留返回值引用属性。

---

## 3-23 C++11 中的 auto 是怎么实现识别自动类型的？模板是怎么实现转化成不同类型的？

### 核心答案
`auto` 的类型推导规则本质上和模板类型推导非常接近。编译器会根据初始化表达式，把 `auto` 当成模板参数来推导具体类型。

### 类比模板
```cpp
template<typename T>
void f(T x);
```

如果调用：
```cpp
int a = 10;
f(a);
```
则 `T` 被推导为 `int`。`auto x = a;` 的推导也类似。

### 推导规则简化版
1. 忽略顶层 `const`
2. 默认不保留引用
3. `auto&` 会保留引用
4. `auto&&` 在推导中可能形成万能引用

### 模板为什么能“适配不同类型”
因为模板本质上不是一个函数，而是一组待实例化的代码模式。编译器在实例化时，为不同类型生成不同代码。

---

## 3-24 你觉得在 auto 和 C++98 中模板之间的关系是什么？

### 核心答案
`auto` 的推导规则可以看作是模板类型推导规则在变量声明场景中的自然延伸。C++98 模板已经具备“根据实参推导类型”的能力，C++11 的 `auto` 把这种能力推广到了普通变量声明。

### 可以这样回答
“我理解 `auto` 并不是一个全新完全独立的系统，它本质上借鉴了模板参数推导规则。模板是在函数 / 类模板实例化时推导类型，`auto` 是在变量声明时让编译器做类似的推导。”

---

## 3-25 C++ 多线程 / C++ 异步

### 多线程
C++11 提供了标准线程库：
- `std::thread`
- `std::mutex`
- `std::condition_variable`
- `std::future`
- `std::async`
- `std::packaged_task`

### 异步
#### 1）`std::async`
把任务异步执行，并通过 `future` 拿结果。

```cpp
auto fut = std::async(std::launch::async, [] { return 42; });
std::cout << fut.get();
```

#### 2）`std::promise` / `std::future`
用于线程间传递结果。

#### 3）线程池 / 事件循环 / 协程
工程里更常见。

### 面试表达
“多线程解决并发执行，异步更强调任务提交与结果获取解耦。工程里常见是线程池 + future/callback，现代场景会进一步结合协程。”

---

## 3-26 条件变量为什么要和 mutex 搭配，不能单独使用吗

### 核心答案
条件变量必须和互斥锁搭配使用，因为等待条件成立涉及两个动作：
1. 检查条件
2. 进入等待状态

如果这两步不是原子完成，就会发生竞态和丢失唤醒。

### 正确模式
```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

void worker() {
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, [] { return ready; });
}
```

### 为什么不能单独用
假设：
- 线程 A 检查到 `ready == false`
- 还没进入等待
- 线程 B 把 `ready` 改成 `true` 并通知
- A 再进入等待，通知已经错过

这就是丢失唤醒。

### 结论
`mutex + condition_variable` 是一个整体协议，不是两个独立工具的偶然组合。

---

## 3-27 讲一下 C++ 中的 forward 函数，以及它是在哪个版本出现的

### 核心答案
`std::forward` 出现在 **C++11**，用于在模板中实现完美转发，保留参数原本的值类别（左值或右值）。

### 为什么需要 `forward`
模板中参数即使声明为 `T&&`，在函数体内只要有名字，它就是左值：

```cpp
template<typename T>
void wrapper(T&& x) {
    foo(x); // x 在这里是左值
}
```

如果要保持调用者传进来的左值 / 右值属性，就要用：

```cpp
foo(std::forward<T>(x));
```

### 典型实现
```cpp
template<class T>
T&& forward(std::remove_reference_t<T>& t) noexcept {
    return static_cast<T&&>(t);
}
```

### 作用总结
- 左值传进来，仍转发为左值
- 右值传进来，仍转发为右值

---

## 3-28 左值引用和右值引用有什么区别

### 核心区别
| 项目 | 左值引用 | 右值引用 |
|---|---|---|
| 写法 | `T&` | `T&&` |
| 绑定对象 | 左值 | 右值 |
| 主要用途 | 普通引用、参数传递 | 移动语义、完美转发 |

### 补充
- `const T&` 可以绑定右值
- 有名字的右值引用变量本身是左值

### 面试回答模板
“左值引用主要用于给已有对象起别名，右值引用主要用于接管临时对象资源，实现移动语义和完美转发。”

---

## 3-29 还有哪些可调用的对象？

### C++ 中常见可调用对象
1. 普通函数
2. 函数指针
3. lambda 表达式
4. 仿函数（重载 `operator()` 的对象）
5. 成员函数指针
6. `std::bind` 绑定结果
7. `std::function` 包装后的可调用对象

### 示例
```cpp
void f();
struct Functor {
    void operator()() {}
};
auto lam = []{};
```

### 面试补充
`std::function` 是“可调用对象的统一封装器”，而不是一种新的可调用对象类型本体。

---

## 3-30 lambda 表达式使用场景

### 常见场景
1. STL 算法
```cpp
std::sort(v.begin(), v.end(), [](int a, int b){ return a < b; });
```
2. 回调函数
3. 线程池提交任务
4. 本地小策略 / 局部逻辑
5. 事件处理
6. 延迟执行

### 为什么常用
- 代码局部化
- 可捕获上下文
- 避免为了小逻辑单独写函数

---

## 3-31 constexpr 与 const 指针的区别

### `const`
表示“只读”，不一定是编译期常量。

```cpp
const int x = getValue(); // 运行期也可能确定
```

### `constexpr`
表示“编译期可求值的常量表达式”，更强。

```cpp
constexpr int x = 10;
```

### 区别
- `const`：只保证不可修改
- `constexpr`：要求编译期可求值

### 对指针的理解
```cpp
const int* p1;        // 指向 const int
int* const p2 = &x;   // const 指针
constexpr int* p3 = nullptr; // 编译期常量指针值
```

### 面试一句话
“`const` 主要是只读语义，`constexpr` 强调编译期常量能力。”

---

## 3-32 join 和 detach 线程的区别

### `join`
- 阻塞等待线程执行完成
- 回收线程资源
- 线程对象和执行线程重新同步

```cpp
std::thread t(work);
t.join();
```

### `detach`
- 线程与 `std::thread` 对象分离
- 后台独立运行
- 无法再 `join`
- 生命周期更难管理

```cpp
std::thread t(work);
t.detach();
```

### 风险
`detach` 容易导致：
- 程序退出时后台线程仍在跑
- 访问已析构对象
- 资源管理困难

### 面试建议
工程里优先 `join` 或线程池；`detach` 要非常谨慎。

---

## 3-33 C++ 协程用过吗

### 标准回答
如果用过：
“用过，主要理解是把异步流程写成类似同步代码的形式，编译器会把协程函数转换成状态机，通过 `co_await`/`co_return`/`co_yield` 控制挂起和恢复。”

如果没真正项目使用过，也可以这样答：
“项目里还没有大规模落地，但我理解它的本质是编译器生成状态机，特别适合高并发 IO 场景，能改善回调地狱。”

### 核心点
- C++20 正式标准化协程
- 关键字：`co_await`、`co_yield`、`co_return`
- 本质：用户态可挂起函数 + 状态机转换

---

## 3-34 final 关键字

### 用途 1：修饰类
表示该类不能再被继承。

```cpp
class Base final {};
```

### 用途 2：修饰虚函数
表示该虚函数不能再被子类重写。

```cpp
class Base {
    virtual void f() final;
};
```

### 作用
1. 明确设计意图
2. 防止错误继承 / 重写
3. 某些场景可帮助优化（编译器更容易去虚拟化）

---

## 3-35 shared_ptr 能想到几种实现方法？

### 核心答案
思路上至少有两类：

### 方案 1：对象和引用计数分离
- 对象单独分配
- 计数器 / 控制块单独分配

### 方案 2：对象和控制块一起分配
- 类似 `make_shared`
- 一次分配内存，控制块和对象放一起
- 更高效

### 控制块中一般包含
- 强引用计数
- 弱引用计数
- 删除器
- 分配器
- 对象地址或对象内嵌空间

### 面试可展开
从工程角度讲：
- 一体分配性能更好
- 分离分配灵活性更高

---

## 3-36 引用计数器是什么样的形式存在

### 核心答案
通常不是直接存在对象里，而是存在 **控制块（control block）** 中。一个 `shared_ptr` 一般持有：
1. 对象指针
2. 控制块指针

### 控制块常见内容
- `strong_count`
- `weak_count`
- 删除器
- 分配器

### 为什么不直接放对象里
1. 原始对象不一定能修改定义
2. 支持自定义删除器和类型擦除
3. 更灵活支持 `weak_ptr`

---

## 3-37 左值，右值，万能引用，完美转发

### 左值 / 右值
见前面 3-11。

### 万能引用
当 `T&&` 出现在模板参数推导场景中时，它不是普通右值引用，而是万能引用 / 转发引用。

```cpp
template<typename T>
void f(T&& x);
```

- 传左值时，`T` 推导成 `U&`
- 传右值时，`T` 推导成 `U`

### 完美转发
通过 `std::forward<T>(x)` 保留参数原本值类别。

```cpp
template<typename T>
void wrapper(T&& x) {
    foo(std::forward<T>(x));
}
```

### 为什么重要
因为包装函数、工厂函数、容器 `emplace` 都依赖它避免把右值错误当左值传递。

---

## 3-38 shared_ptr 线程安全吗

### 核心答案
**引用计数的增减是线程安全的，但对象本身不是自动线程安全的。**

### 具体解释
#### 线程安全的部分
多个线程分别拷贝 / 析构不同的 `shared_ptr` 实例，控制块中的计数通常通过原子操作维护，这是安全的。

#### 不线程安全的部分
如果多个线程同时访问 `shared_ptr` 所指向的对象内容，仍然需要额外同步。

### 一句话总结
`shared_ptr` 解决的是“生命周期共享安全”，不是“对象内部状态访问安全”。

---

## 3-39 push_back 左值和右值的区别

### 核心答案
容器的 `push_back` 一般有左值和右值两个重载：
- 传左值：调用拷贝构造
- 传右值：调用移动构造

### 示例
```cpp
std::vector<std::string> v;
std::string s = "hello";
v.push_back(s);              // 拷贝
v.push_back(std::move(s));   // 移动
v.push_back("world");       // 右值，通常移动 / 构造
```

### 意义
右值版本能减少深拷贝，性能更好。

### 面试追问
`emplace_back` 呢？  
`emplace_back` 可以直接原地构造对象，进一步减少中间临时对象。

---

## 3-40 std::forward 源码分析，讲一下 C++ 中的 forward 函数，以及它是在哪个版本出现的

### 版本
**C++11**。

### 核心作用
实现完美转发，保留值类别。

### 简化源码
```cpp
template<typename T>
T&& forward(std::remove_reference_t<T>& t) noexcept {
    return static_cast<T&&>(t);
}
```

### 分析
- 如果 `T` 是左值引用类型，`T&&` 折叠后仍是左值引用
- 如果 `T` 是非引用类型，`T&&` 就是右值引用

### 为什么需要它
模板函数内参数有名字，所以表达式本身是左值，不转发的话会丢失右值属性。

---

## 3-41 还有 explicit，移动语义，完美转发源码实现和移动语义底层实现

### `explicit`
用于禁止构造函数或转换运算符发生隐式转换。

```cpp
class A {
public:
    explicit A(int x) {}
};
```

### 为什么需要
避免：
```cpp
void foo(A a);
foo(10); // 如果构造函数不是 explicit，可能发生隐式转换
```

### 移动语义底层实现
底层还是靠：
- 右值引用
- 资源转移
- 源对象置空

### 完美转发源码实现
核心是：
- 模板推导
- 引用折叠
- `std::forward`

### 面试回答建议
这题很散，最好分三段说：
1. `explicit` 的作用
2. 移动语义是资源所有权转移
3. 完美转发依赖 `T&& + 引用折叠 + forward`

---

## 3-42 结构化绑定、移动语义、智能指针。unique_ptr 独占的实现

### 结构化绑定
C++17 特性，可把 tuple / pair / struct 拆开：

```cpp
auto [x, y] = std::make_pair(1, 2);
```

### 和移动语义关系
结构化绑定默认可能发生拷贝，也可以配合引用：

```cpp
auto& [a, b] = p;
```

### 与智能指针关系
可以结构化绑定返回多个对象，但 `unique_ptr` 仍然遵循独占所有权规则，不能拷贝。

### `unique_ptr` 独占实现核心
```cpp
template<typename T>
class unique_ptr {
    T* ptr;
public:
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;

    unique_ptr(unique_ptr&& other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }
};
```

### 面试重点
独占的关键不是“文档说它独占”，而是“语言层面删除了拷贝构造和拷贝赋值”。

---

## 3-43 lambda 参数捕获有几种方式

### 捕获方式
- `[]`
- `[=]`
- `[&]`
- `[x]`
- `[&x]`
- `[this]`
- `[*this]`

### 补充
lambda 还有参数列表，不要把“参数传递”和“捕获”混淆。

```cpp
int a = 10;
auto f = [a](int x) { return a + x; };
```

这里：
- `a` 是捕获
- `x` 是参数

---

## 3-44 C++20 新特性：主要介绍协程

### 可答的重点
1. 协程标准化进入 C++20
2. 关键字：`co_await`、`co_yield`、`co_return`
3. 编译器把协程函数转成状态机
4. 特别适合异步 IO、高并发场景

### 相比线程
- 线程是操作系统调度实体
- 协程是用户态轻量挂起 / 恢复机制
- 协程切换开销更小

### 面试表达模板
“C++20 协程的核心价值是把异步流程写成接近同步的风格，避免回调地狱。底层编译器会把协程函数转成状态机，适合事件驱动和高并发 IO 场景。”

---

## 3-45 如何实现共享指针

### 核心思路
实现一个简化版 `shared_ptr` 至少需要：
1. 对象指针
2. 引用计数器
3. 拷贝时计数加一
4. 析构时计数减一
5. 计数归零时释放对象和计数器

### 简化示例
```cpp
template<typename T>
class SharedPtr {
public:
    explicit SharedPtr(T* p = nullptr) : ptr_(p), cnt_(new size_t(1)) {}

    SharedPtr(const SharedPtr& other) : ptr_(other.ptr_), cnt_(other.cnt_) {
        ++(*cnt_);
    }

    ~SharedPtr() {
        if (--(*cnt_) == 0) {
            delete ptr_;
            delete cnt_;
        }
    }

private:
    T* ptr_;
    size_t* cnt_;
};
```

### 工业级实现还需要
- 线程安全
- 弱引用计数
- 自定义删除器
- 异常安全
- 控制块

---

## 3-46 weak_ptr 真的不计数？是否有计数方式，在哪分配的空间

### 核心答案
`weak_ptr` **不增加强引用计数**，但通常会增加 **弱引用计数**。所以它不是“完全不计数”，而是“不参与对象是否存活的强计数”。

### 控制块里通常有
- 强引用计数 `strong_count`
- 弱引用计数 `weak_count`

### 分配位置
一般在堆上的控制块中分配。`shared_ptr` 和 `weak_ptr` 都指向同一个控制块。

### 对象和控制块释放条件
- 强计数归零：对象析构
- 强计数和弱计数都归零：控制块释放

---

## 3-47 为什么会引入完美转发

### 核心答案
因为模板包装层会把右值参数“变成左值”，导致下层函数无法感知调用者原本传的是右值，从而失去移动语义和最优重载匹配。

### 问题示例
```cpp
template<typename T>
void wrapper(T&& x) {
    foo(x); // x 有名字，是左值
}
```

### 完美转发的目标
- 调用者传左值，继续传左值
- 调用者传右值，继续传右值

### 解决手段
`std::forward<T>(x)`

### 为什么重要
没有完美转发，泛型工厂、容器 `emplace`、线程池提交接口都没法优雅高效地工作。

---

## 3-48 智能指针介绍下，weak_ptr 底层实现原理

### 智能指针总览
- `unique_ptr`：独占
- `shared_ptr`：共享
- `weak_ptr`：弱观察

### `weak_ptr` 底层实现原理
`weak_ptr` 通常只持有控制块引用，不拥有对象。

#### 关键点
1. 它和 `shared_ptr` 指向同一个控制块
2. 构造 / 拷贝时增加弱引用计数
3. 不增加强引用计数
4. `lock()` 时检查强引用计数是否大于 0
   - 大于 0：生成新的 `shared_ptr`
   - 等于 0：返回空

### 面试一句话
“`weak_ptr` 的底层关键不是对象指针本身，而是共享控制块中的弱计数和 `lock()` 检查逻辑。”

---

## 3-49 写一个 unique_ptr

### 简化版实现
```cpp
template<typename T>
class UniquePtr {
public:
    explicit UniquePtr(T* p = nullptr) : ptr_(p) {}

    ~UniquePtr() { delete ptr_; }

    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;

    UniquePtr(UniquePtr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }

    UniquePtr& operator=(UniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr_;
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }

    T* release() {
        T* tmp = ptr_;
        ptr_ = nullptr;
        return tmp;
    }

    void reset(T* p = nullptr) {
        delete ptr_;
        ptr_ = p;
    }

private:
    T* ptr_;
};
```

### 面试加分点
还能继续提：
- 自定义删除器
- 数组特化
- `swap`
- `make_unique`

---

## 3-50 override 和 final

### `override`
表示当前虚函数是重写父类虚函数。编译器会检查签名是否真正匹配。

```cpp
class Base {
public:
    virtual void f();
};

class Derived : public Base {
public:
    void f() override;
};
```

### `final`
见 3-34，可修饰类或虚函数，禁止继续继承 / 重写。

### 二者一起回答
- `override`：防止误写错签名
- `final`：明确禁止进一步改写

---

## 3-51 智能指针，有没有释放不了的情况？

### 有，典型有以下几类

#### 1）循环引用
两个对象互相 `shared_ptr`。

#### 2）回调闭包形成逻辑环
对象持有回调，回调又按值捕获 `shared_ptr<this>`。

#### 3）错误的自定义删除器
删除器什么都不做，或释放方式错误。

#### 4）同一裸指针被多个控制块管理
可能不是“不释放”，而是错误释放。

#### 5）全局 / 静态对象长期持有
看起来像泄漏，实际是程序生命周期内一直持有。

### 面试总结
智能指针不是万能的，核心仍是**所有权设计**。设计错了，智能指针也救不了。

---

## 3-52 智能指针是如何引用计数自动加一

### 核心答案
当 `shared_ptr` 发生拷贝构造或拷贝赋值时，会共享同一个控制块，并对控制块中的强引用计数执行加一。

### 伪代码
```cpp
SharedPtr(const SharedPtr& other)
    : ptr_(other.ptr_), ctrl_(other.ctrl_) {
    ++ctrl_->strong_count;
}
```

### 自动的含义
不是编译器魔法，而是类的拷贝构造 / 赋值运算符重载里写好了这套逻辑。

### 线程安全版本
工业实现里一般用原子加一。

---

## 3-53 现代 C++ 函数式编程

### 核心理解
现代 C++ 的函数式编程并不是像 Haskell 那样纯函数式，而是借助：
- lambda
- 高阶函数
- STL 算法
- 不可变思维
- `std::function`
- `bind`
- ranges（C++20）

来让代码更声明式。

### 常见写法
```cpp
std::vector<int> v{1,2,3,4};
std::for_each(v.begin(), v.end(), [](int x){ std::cout << x; });
```

### 现代趋势
C++20 `ranges` 进一步增强了组合式、管道式风格。

### 面试可说
“我理解现代 C++ 函数式编程主要体现在 lambda、高阶算法、不可变数据倾向和 ranges 风格，目标是提升表达力和组合能力，而不是完全纯函数式。”

---

## 3-54 override 和 overload 关系是什么

### `override`
重写，发生在继承体系中，要求基类有虚函数，函数签名匹配。

### `overload`
重载，同一作用域中函数名相同、参数列表不同。

```cpp
void f(int);
void f(double); // overload
```

### 区别
- `override`：父子类、多态相关
- `overload`：同作用域、编译期重载解析

### 常见误区
重写时如果签名不一致，可能不是 override，而是隐藏了父类函数。

---

## 3-55 智能指针和裸指针混用的问题

### 核心答案
智能指针和裸指针可以混用，但必须非常明确所有权边界。最大风险是：
1. 重复释放
2. 悬空指针
3. 生命周期错判

### 错误示例 1：手动 delete 智能指针管理的裸指针
```cpp
auto sp = std::make_shared<Foo>();
Foo* raw = sp.get();
delete raw; // 错误，sp 后续还会再释放一次
```

### 错误示例 2：同一裸指针交给多个智能指针
```cpp
Foo* p = new Foo;
std::shared_ptr<Foo> a(p);
std::shared_ptr<Foo> b(p); // 错，两个控制块
```

### 正确混用原则
- 裸指针只做借用，不做所有权表达
- 不要对 `get()` 出来的裸指针 `delete`
- 如果要从 `unique_ptr` 交给别人，明确用 `release()` / `std::move`

### 面试一句话
“智能指针和裸指针混用不是绝对禁止，关键是裸指针不能偷偷承担所有权，否则就会和智能指针的资源管理发生冲突。”

---

# 最后复习建议

如果你要把这份文档拿去面试前背，建议优先分成 6 个专题复习：
1. 智能指针与 RAII
2. 左值右值 / 移动语义 / `std::move`
3. 完美转发 / `std::forward`
4. 锁、条件变量、原子
5. lambda / `auto` / `decltype`
6. shared_ptr / weak_ptr 底层与典型陷阱

如果你下一步想继续，我可以再帮你做两版：
- **背诵版**：每题压缩成 30 秒 / 1 分钟口述答案
- **进阶版**：每题补更多源码分析、控制块图示、线程模型和面试追问链
