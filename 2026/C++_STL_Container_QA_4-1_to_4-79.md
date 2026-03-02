# C++ STL / 容器 高频面试题（4-1 ~ 4-79）详细解答

> 适用方向：C++ 后端 / 基础架构 / 中间件 / 自动驾驶 / 高性能服务  
> 写法目标：尽量覆盖 **概念 + 底层 + 时间复杂度 + 代码示例 + 面试追问点**

---

## 4-1 `unordered_map` 和 `map` 区别

### 核心区别
- `map`
  - **底层**：通常是红黑树（平衡二叉搜索树）
  - **有序**
  - 查找 / 插入 / 删除：`O(log n)`
- `unordered_map`
  - **底层**：哈希表（桶数组 + 冲突处理）
  - **无序**
  - 平均查找 / 插入 / 删除：`O(1)`，最坏 `O(n)`

### 什么时候用
- 需要**有序遍历**、范围查询、`lower_bound/upper_bound`：选 `map`
- 只追求**快速查找**：选 `unordered_map`

### 示例
```cpp
std::map<int, std::string> mp;
mp[3] = "c";
mp[1] = "a";
mp[2] = "b";
// 遍历顺序: 1 2 3

std::unordered_map<int, std::string> ump;
ump[3] = "c";
ump[1] = "a";
ump[2] = "b";
// 遍历顺序不固定
```

### 追问点
- `unordered_map` 为什么最坏 `O(n)`？  
  因为大量元素哈希到同一个桶里，退化成链表/长桶扫描。
- `map` 的稳定性通常比 `unordered_map` 更好，因为复杂度更稳定。

---

## 4-2 `vector` 和 `list` 区别；链表和数组差异

### `vector`
- 连续内存
- 支持随机访问：`O(1)`
- 尾插均摊 `O(1)`
- 中间插入/删除要搬移元素：`O(n)`
- cache locality 好，遍历快

### `list`
- 双向链表
- 不支持随机访问
- 已知位置插入/删除 `O(1)`
- 每个节点额外存前驱/后继指针，内存开销大
- cache locality 差，遍历慢

### 数组 vs 链表
- 数组：连续、随机访问快、插删慢
- 链表：非连续、插删快、访问慢

### 面试观点
现代 CPU 下，**很多场景 `vector` 实际上比 `list` 更快**，哪怕 `list` 理论插删复杂度更好，因为缓存命中率差别很大。

---

## 4-3 `deque` 的实现

### 核心
`deque` 不是完全连续内存，也不是普通链表。  
通常实现是：

- 一张 **map（控制表）**
- map 里每个元素指向一块固定大小的缓冲区（buffer）
- 多块 buffer 组合成逻辑上的双端队列

### 特点
- 头尾插入删除都比较高效
- 随机访问也支持，但比 `vector` 慢一点
- 不像 `vector` 那样必须整体连续，因此头插比 `vector` 强很多

### 简化理解
`deque = 分段连续数组`

---

## 4-4 `vector` 扩容后只使用了一小部分，怎么释放后面的空间

### 方法 1：`shrink_to_fit()`
```cpp
std::vector<int> v(100000);
v.resize(10);
v.shrink_to_fit(); // 请求释放多余 capacity
```

注意：`shrink_to_fit()` **只是非强制请求**，标准不保证一定释放。

### 方法 2：swap 技巧（更常用）
```cpp
std::vector<int>(v).swap(v);
```

如果只保留前 10 个元素：
```cpp
v.resize(10);
std::vector<int>(v).swap(v);
```

---

## 4-5 `vector` 删除值为偶数

### 写法 1：erase-remove 惯用法（推荐）
```cpp
std::vector<int> v{1,2,3,4,5,6,7,8};

v.erase(std::remove_if(v.begin(), v.end(),
                       [](int x) { return x % 2 == 0; }),
        v.end());
```

### 原理
- `remove_if` 只是把“不删除的元素”往前搬，返回新的逻辑结尾
- `erase` 真正删除尾部多余元素

### 复杂度
整体 `O(n)`

---

## 4-6 `vector` 和 `array` 区别

### `std::array`
- 固定大小
- 封装原生数组
- 大小是类型的一部分：`std::array<int, 10>`
- 不扩容

### `std::vector`
- 动态大小
- 可扩容
- 底层连续内存

### 使用建议
- 大小编译期已知且固定：`std::array`
- 大小运行期变化：`std::vector`

---

## 4-7 常用 STL

### 常见容器
- 顺序容器：`vector`、`deque`、`list`、`array`
- 关联容器：`map`、`set`、`multimap`、`multiset`
- 无序容器：`unordered_map`、`unordered_set`
- 适配器：`stack`、`queue`、`priority_queue`

### 常用算法
- `sort`、`find`、`count`
- `remove_if`、`transform`
- `lower_bound`、`upper_bound`

### 常用工具
- `pair`、`tuple`
- `function`
- `bind`
- 智能指针

---

## 4-8 常用 STL 底层实现

- `vector`：动态数组
- `list`：双向链表
- `deque`：分段连续数组
- `map/set`：红黑树
- `unordered_map/set`：哈希表
- `priority_queue`：堆（通常基于 `vector`）
- `stack/queue`：容器适配器，默认分别基于 `deque`

---

## 4-9 Map 的种类

- `map`：key 唯一，有序
- `multimap`：key 可重复，有序
- `unordered_map`：key 唯一，无序
- `unordered_multimap`：key 可重复，无序

---

## 4-10 `vector` 与 `list` 区别，`map` 如何实现，查找效率多少

### `vector` vs `list`
同 4-2。

### `map`
- 底层：红黑树
- 查找、插入、删除：`O(log n)`

### 红黑树特点
- 近似平衡
- 避免退化成链表
- 高度 `O(log n)`

---

## 4-11 `vector` 和 `list` 的区别？无序怎么查找一个数

### 无序查找
如果只是普通 `vector/list` 无序存放元素：
- `std::find(begin, end, x)`：顺序查找，`O(n)`

### 说明
不是“list 不能用 find”，而是：
- `list` 可以用 `std::find`
- 只是依然是顺序扫描，不能随机访问，也不会更快

```cpp
auto it = std::find(lst.begin(), lst.end(), 10);
```

---

## 4-12 `map` 用什么实现；能不能用别的实现；哈希冲突解决方式

### `map`
- 通常是红黑树

### 能不能用别的实现
可以，理论上也可以用：
- AVL 树
- Treap
- B 树 / B+ 树（数据库常见）
- 跳表

但标准库里 `map` 通常选红黑树，因为插入/删除/查找综合性能和旋转成本比较均衡。

### 哈希冲突解决方式
- 拉链法（链地址法）
- 开放寻址法
  - 线性探测
  - 二次探测
  - 双重哈希

C++ STL 的 `unordered_map` 主流实现通常是 **拉链法**。

---

## 4-13 `vector` / `list` 区别
同 4-2，可直接复用。

---

## 4-14 `map` 和 `hashmap` 的区别

这里的 `hashmap` 通常指 `unordered_map`。

- `map`：红黑树，有序，`O(log n)`
- `unordered_map`：哈希表，无序，平均 `O(1)`

### 额外差别
- `map` 可做范围查询
- `unordered_map` 不支持 `lower_bound`

---

## 4-15 `std::copy` 应用

### 用法
```cpp
std::vector<int> a{1,2,3};
std::vector<int> b(3);

std::copy(a.begin(), a.end(), b.begin());
```

### 注意
目标区间必须足够大，否则越界。

### 插入式写法
```cpp
std::vector<int> b;
std::copy(a.begin(), a.end(), std::back_inserter(b));
```

### 适用
- 拷贝数组到容器
- 拷贝容器到容器
- 配合迭代器做泛型复制

---

## 4-16 `vector` 扩容是 1.5 倍还是 2 倍

### 标准结论
**标准没有强制规定增长倍数**。  
具体由实现决定，常见策略：
- GCC/libstdc++：常见接近 2 倍或按策略增长
- MSVC：常见接近 1.5 倍

### 目的
在减少扩容次数和避免浪费太多空间之间折中。

---

## 4-17 `vector` 插入的时间复杂度

- 尾部 `push_back`
  - 若 capacity 足够：`O(1)`
  - 若触发扩容：均摊 `O(1)`，单次最坏 `O(n)`
- 中间/头部插入
  - 需要搬移后面的元素：`O(n)`

---

## 4-18 `vector` 的实现机制；类对象扩容会不会析构；`remove` 如何实现

### 实现机制
底层三块核心状态：
- `begin`
- `end`
- `end_of_storage`

### 扩容时会做什么
1. 申请更大内存
2. 把旧元素搬过去（拷贝或移动构造）
3. 销毁旧空间里的对象
4. 释放旧内存

### 如果元素是类对象，会不会析构
**会**。旧内存中的对象在迁移后会被析构。

### `remove`
`std::remove` 并不真正删除，而是：
- 把不该删的元素向前覆盖
- 返回新的逻辑末尾

```cpp
auto it = std::remove(v.begin(), v.end(), 2);
v.erase(it, v.end());
```

---

## 4-19 `vector` 的内存以什么方式增大

通过 **重新分配更大连续内存** 来增大。

不是在原内存块后面直接拼接，因为连续空间未必可用。

---

## 4-20 迭代器失效

### 常见规则
#### `vector`
- 扩容后：所有迭代器、引用、指针基本都失效
- erase 后：被删位置及之后迭代器失效

#### `list`
- 插入通常不失效
- 删除某节点，只会使该节点迭代器失效

#### `map/set`
- 插入一般不失效
- 删除某节点，只使该节点迭代器失效

#### `unordered_map`
- rehash 后可能全部迭代器失效

---

## 4-21 `vector` 有没有右值引用优化 `push_back` 接口

有。

```cpp
void push_back(const T& value); // 左值
void push_back(T&& value);      // 右值
```

右值版本可触发移动构造，减少拷贝。

另外还有：
```cpp
template<class... Args>
reference emplace_back(Args&&... args);
```

---

## 4-22 `map` 和 `set` 插入删除有啥区别

### 相同点
- 都通常基于红黑树
- 插入/删除/查找复杂度都为 `O(log n)`

### 不同点
- `set` 存的是键本身
- `map` 存的是 `pair<const Key, T>`

### 语义差异
- `set` 更像“去重集合”
- `map` 是“键值映射”

---

## 4-23 迭代器失效；容器怎么删除元素；`erase` 返回值是什么

### 删除元素一般写法
#### `vector`
```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);
    else ++it;
}
```

### 为什么这样写
因为 `erase` 后当前迭代器失效，`erase` 返回“下一个有效迭代器”。

### `erase` 返回值
- 返回被删元素后面的那个迭代器
- 如果删到最后，返回 `end()`

---

## 4-24 简述 `map`、`unordered_map`；自定义类型当键要注意什么

### 自定义类型做 `map` 的 key
要提供比较规则，通常是 `operator<` 或自定义比较器。

### 自定义类型做 `unordered_map` 的 key
要提供：
1. 哈希函数
2. 相等判断函数

```cpp
struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

struct PointHash {
    size_t operator()(const Point& p) const {
        return std::hash<int>()(p.x) ^ (std::hash<int>()(p.y) << 1);
    }
};

std::unordered_map<Point, int, PointHash> mp;
```

---

## 4-25 `map` 和 `unordered_map` 的底层实现、区别、复杂度、场景
这个问题可直接参考 4-1 和 4-14，整理回答：

- `map`：红黑树，`O(log n)`，有序，适合范围查询
- `unordered_map`：哈希表，平均 `O(1)`，无序，适合高频精确查找

---

## 4-26 `map` 插入一个元素的时间复杂度

- 查找插入位置：`O(log n)`
- 调整树平衡：`O(log n)` 范围内常数复杂度旋转

综合：`O(log n)`

---

## 4-27 实现 `Vector` 类：初始化、整体拷贝、`push_back`、`pop_back`、`operator[]`

下面给一个简化版思路：

```cpp
template<typename T>
class MyVector {
public:
    MyVector() : data_(nullptr), size_(0), capacity_(0) {}

    MyVector(const MyVector& other) {
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_ = new T[capacity_];
        for (size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
    }

    MyVector& operator=(const MyVector& other) {
        if (this == &other) return *this;
        delete[] data_;
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_ = new T[capacity_];
        for (size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
        return *this;
    }

    ~MyVector() { delete[] data_; }

    void push_back(const T& val) {
        if (size_ == capacity_) expand();
        data_[size_++] = val;
    }

    void pop_back() {
        if (size_ > 0) --size_;
    }

    T& operator[](size_t pos) { return data_[pos]; }
    const T& operator[](size_t pos) const { return data_[pos]; }

private:
    void expand() {
        size_t newCap = capacity_ == 0 ? 1 : capacity_ * 2;
        T* newData = new T[newCap];
        for (size_t i = 0; i < size_; ++i) newData[i] = data_[i];
        delete[] data_;
        data_ = newData;
        capacity_ = newCap;
    }

private:
    T* data_;
    size_t size_;
    size_t capacity_;
};
```

### 真正完整实现还要考虑
- 异常安全
- 移动语义
- placement new
- 析构控制
- allocator

---

## 4-28 `std::map` 数据量很大查询性能变差，换什么结构

### 看需求
- 如果不要求有序：考虑 `unordered_map`
- 如果是磁盘/大规模范围检索：考虑 B+ 树
- 如果读多写少：考虑排序 `vector` + 二分查找
- 如果 key 连续：考虑数组/位图

### 面试答法
“是否替换成 `unordered_map` 取决于业务是否需要有序性和范围查询。如果只做精确 key 查找，`unordered_map` 平均复杂度更优；如果还需要区间查询、按序遍历，`map` 仍然更合适。”

---

## 4-29 STL 是线程安全的吗

### 结论
**标准 STL 容器默认不是线程安全的。**

### 更准确说法
- 多线程只读：通常可以
- 一旦有写，或者一个线程写一个线程读：不安全，需同步

---

## 4-30 `map` 并发安全吗？为什么

不安全。

### 原因
- 红黑树节点插入/删除会修改树结构
- 多线程同时改会导致数据竞争、结构损坏
- 一个线程改、一个线程读也可能读到中间状态

### 解决
- 加锁
- 读写锁
- 分片锁
- 并发 map（如 TBB）

---

## 4-31 STL 中顺序数据结构 `vector` `array` `deque`

### `array`
- 固定大小
- 连续
- 零额外动态分配

### `vector`
- 动态大小
- 连续
- 扩容需要重新分配

### `deque`
- 动态大小
- 分段连续
- 头尾操作强

---

## 4-32 `push_back` 和 `emplace_back` 的区别

### `push_back`
传入一个对象，再拷贝/移动进容器。

```cpp
v.push_back(std::string("abc"));
```

### `emplace_back`
直接在容器尾部原地构造对象。

```cpp
v.emplace_back("abc");
```

### 优势
对复杂对象可少一次临时对象构造/移动。

---

## 4-33 大量数据 `vector` 扩容拷贝太重，怎么高效可控

### 方案
1. **提前 `reserve()`**
```cpp
v.reserve(1000000);
```
2. 已知上限时一次分配够
3. 使用 `deque` 避免整体搬迁
4. 如果是大对象，可存指针 / `unique_ptr`
5. 采用分块结构（chunked vector）

---

## 4-34 迭代器的实现

### 本质
迭代器是一个“泛化指针”，封装了访问容器元素的方式。

### 不同容器实现不同
- `vector` 迭代器：本质接近原生指针
- `list` 迭代器：本质是节点指针 + 运算封装
- `map` 迭代器：本质是树节点指针 + 中序遍历逻辑

### 作用
统一算法接口，使 `sort/find/copy` 能作用于不同容器。

---

## 4-35 `vector::push_back` 实现原理；尾部操作后对迭代器影响

### 不扩容
- 在尾部构造一个新元素
- 原有迭代器通常仍然有效
- 但 `end()` 会变化

### 扩容
- 申请新内存
- 迁移元素
- 释放旧内存
- 所有旧迭代器、引用、指针失效

---

## 4-36 STL 库 `vector` 对内存优化

- 几何扩容，减少频繁分配
- 连续内存，提高缓存命中
- `reserve()` 预分配
- 移动语义减少扩容搬迁成本
- allocator 定制内存分配策略

---

## 4-37 `vector` 存储的内容可以是引用吗？指针呢

### 引用
不能直接存引用类型：
```cpp
std::vector<int&> v; // ❌ 不合法
```

因为引用不是可重新绑定的普通对象。

### 指针
可以：
```cpp
std::vector<int*> v;
```

### 替代方案
- `std::reference_wrapper<T>`
```cpp
std::vector<std::reference_wrapper<int>> v;
```

---

## 4-38 新建一个 `vector` 空间大小

### 指定 size
```cpp
std::vector<int> v(10); // size=10, capacity>=10
```

### 指定 size 和初值
```cpp
std::vector<int> v(10, 5);
```

### 只预留 capacity
```cpp
std::vector<int> v;
v.reserve(10); // size=0, capacity>=10
```

---

## 4-39 `vector` 插入元素所做的操作

### 若空间足够
- 末尾构造（尾插）
- 或中间位置后的元素后移，再插入

### 若空间不足
- 分配更大内存
- 搬移旧元素
- 插入新元素
- 销毁旧元素并释放旧空间

---

## 4-40 `vector`、数组和普通数组的区别（为何能追加元素）

### 原生数组
- 固定长度
- 大小不可变
- 无成员函数

### `vector`
- 内部维护 `size` 和 `capacity`
- capacity 不够时自动扩容
- 所以看起来“可以追加元素”

本质上不是原地追加，而是可能**重新分配更大的连续空间**。

---

## 4-41 `vector` 如何判断应该扩容（`size` 和 `capacity`）

### 规则
- `size < capacity`：还能继续放
- `size == capacity`：下一次插入就要扩容

---

## 4-42 `vector::clear()` 是清空所有元素还是清空内存

### 结论
`clear()`：
- 删除所有元素（`size` 变成 0）
- **不保证释放 capacity**

```cpp
std::vector<int> v(1000);
v.clear(); 
// size=0, capacity 通常仍然较大
```

---

## 4-43 如何真正清空 `vector` 内存

### 常见写法
```cpp
std::vector<int>().swap(v);
```

或者：
```cpp
std::vector<int> tmp;
v.swap(tmp);
```

这样通常能释放底层内存。

---

## 4-44 `vector` 如何释放内存空间？怎么写
```cpp
v.clear();
std::vector<int>(v).swap(v); // 或 shrink_to_fit
```

---

## 4-45 `std::array`

### 特点
- 固定大小
- 连续内存
- 封装原生数组
- 支持 STL 接口：`size()`, `begin()`, `end()`

### 示例
```cpp
std::array<int, 3> arr = {1, 2, 3};
```

---

## 4-46 `push_back` 与 `emplace_back` 区别、场景、线程安全

### 区别
- `push_back`：传对象
- `emplace_back`：传构造参数，原地构造

### 线程安全吗
都**不是线程安全的**。

### 如何保证线程安全
1. 加锁
2. 单线程 owner + 消息队列
3. 分片容器
4. 无锁队列替代（如果场景是尾部生产消费）

---

## 4-47 STL 的 `sort` 是什么排序

### 结论
`std::sort` 通常实现为 **Introsort（内省排序）**
- 快速排序为主
- 递归过深时退化到堆排序
- 小区间用插入排序

### 复杂度
平均/最坏都能保证较优，通常 `O(n log n)`

---

## 4-48 `unordered_map` 会发生哈希冲突吗？怎么解决，怎么避免

### 会发生
不同 key 可能映射到同一桶。

### 解决方式
- 拉链法
- 开放寻址法

### C++ STL 主流
通常是 **拉链法**

### 如何减少冲突
- 选更好的哈希函数
- 合理设置桶数量
- 控制装载因子（load factor）
- 提前 `reserve()`

---

## 4-49 迭代器和指针对比

### 相同
- 都能间接访问对象：`*it`
- 都能前进：`++it`

### 不同
- 指针只适用于连续内存
- 迭代器是抽象接口，适配各种容器
- 某些迭代器支持随机访问，某些只支持双向/单向

---

## 4-50 看过 `unordered_map` 源码吗？什么时候再哈希

### 再哈希触发条件
当元素数量增大，导致：
```cpp
load_factor() > max_load_factor()
```
就可能触发 rehash。

### rehash 做什么
- 申请更多桶
- 按新桶数重新计算每个元素位置
- 把节点重新挂到新桶里

---

## 4-51 `set` 实现原理

### 底层
通常是红黑树。

### 特点
- key 唯一
- 自动有序
- 插入/删除/查找：`O(log n)`

---

## 4-52 自己实现哈希表 + 拉链法，多线程无锁会有哪些问题

### 问题
1. 多线程同时插入同一桶
   - 可能链表断裂
   - 节点丢失
2. 一个线程 rehash，另一个线程读
   - 读到旧桶 / 空桶 / 中间状态
3. 可见性问题
   - 写线程插入了节点，读线程还没看到
4. ABA / 指针悬挂
   - 节点被改动、释放

### “有元素 C，但 get() 不到” 的原因
- 未同步导致可见性问题
- 正在 rehash
- 链表头更新竞争丢失
- 内存重排序

### 不加锁怎么解决
- 使用原子指针 + CAS
- Hazard Pointer / RCU / epoch-based reclamation
- lock-free hash map 设计

这个已经进入高级并发数据结构范畴。

---

## 4-53 `vector` 扩容用的是 `new` 还是 `malloc`

### 标准库层面
通常通过 **allocator**
默认 allocator 最终一般会调用 `operator new`

### 不是直接简单 `malloc`
因为对象构造/析构需要区分：
- 分配原始内存
- 在内存上构造对象（placement new）

---

## 4-54 `HashMap` 什么时候 rehash？细节

### 触发
元素数 / 桶数 > 最大装载因子

### 细节步骤
1. 计算新桶数
2. 分配新桶数组
3. 遍历旧桶
4. 对每个节点重新计算桶下标
5. 接到新桶中
6. 释放旧桶数组

### 面试追问
rehash 为什么耗时大？  
因为要遍历所有元素并重新分布，复杂度接近 `O(n)`。

---

## 4-55 `vector` 内存优化，动态扩容如何提高效率

### 核心优化点
- 提前 `reserve`
- 使用移动语义减少拷贝
- 避免频繁缩容扩容抖动
- 存大对象时可考虑存指针或索引
- 批量插入时一次预留空间

---

## 4-56 比较器如何判断相同

### 对于 `set/map`
“相同”不是 `==`，而是：

如果
```cpp
!comp(a, b) && !comp(b, a)
```
则认为 `a` 和 `b` 等价。

这点非常重要。

---

## 4-57 `vector` 底层怎么优化，当头一棒

你可以这么答：

### 优化点
1. 连续内存，缓存友好
2. 几何扩容，降低扩容次数
3. 支持移动语义
4. allocator 可定制
5. 小对象尾插非常快

### 如果面试官问“头插为什么差”
因为头插要搬移后面所有元素。

---

## 4-58 `map` 和 `set` 的区别和底层实现，`find` / `[]` / `at`

### `map`
- 存键值对
- `find(key)`：查找，不插入
- `operator[](key)`：如果 key 不存在，会**插入默认值**
- `at(key)`：如果不存在，抛异常 `out_of_range`

### `set`
- 只存 key，没有 `[]`

---

## 4-59 STL 常见容器，allocator 原理，不同大小调用不同函数？

### STL allocator
负责：
- 原始内存分配
- 对象构造
- 对象析构
- 原始内存释放

### 经典 SGI STL 两级配置器思想
- 小块内存：内存池管理
- 大块内存：直接走系统分配

现代实现细节会不同，但面试时提两级配置器是加分点。

---

## 4-60 `unordered_map` 底层实现、冲突解决、扩容机制

### 底层
- 桶数组
- 每个桶挂链表 / 节点

### 冲突解决
- STL 主流实现通常是链地址法

### 扩容
- 超过装载因子后 rehash
- 桶数增大
- 所有元素重新分桶

---

## 4-61 迭代器是干嘛的，如何实现

### 干嘛的
给算法提供统一访问接口，让不同容器都能用相同算法。

### 如何实现
- `vector`：像指针
- `list`：封装节点跳转
- `map`：封装树中序遍历

---

## 4-62 STL 应用：写一个 `vector` 删除偶数节点

```cpp
std::vector<int> v{1,2,3,4,5,6};

v.erase(std::remove_if(v.begin(), v.end(),
                       [](int x){ return x % 2 == 0; }),
        v.end());
```

---

## 4-63 算尾插 n 个数到 `vector` 比原生数组慢多少（拷贝多少次）

### 原生数组
如果预先分配够大，直接写入，无扩容搬迁。

### `vector`
如果不 `reserve`，扩容时会有额外搬迁。

假设按 2 倍扩容，插入 n 个元素时，历史搬迁总次数约是：
- `1 + 2 + 4 + 8 + ... < n`
- 所以总额外搬迁 < `n`

因此总复杂度仍是均摊线性级别。  
可以说：**均摊下来每次插入仍近似 O(1)**，但常数比原生数组更大。

---

## 4-64 `push_back` 和 `emplace_back`
同 4-32 / 4-46。

---

## 4-65 `vector` 动态扩容

### 流程
1. 判断 `size == capacity`
2. 分配更大空间
3. 迁移旧元素
4. 析构旧对象
5. 释放旧内存
6. 更新指针和 capacity

---

## 4-66 `vector` 存入指针和 `int` 有什么区别

### `int`
- 值语义
- 扩容搬迁时拷贝/移动的是整数值

### 指针
- 存的是地址
- 扩容搬迁时复制地址本身，不会复制指针指向的对象

### 风险
- 容器销毁不会自动 `delete` 原始指针指向对象
- 容易内存泄漏 / 悬空

推荐：
```cpp
std::vector<std::unique_ptr<Foo>>
```

---

## 4-67 哈希表算出位置却没插到那个位置，为什么

### 因为冲突
哈希值算出的桶位置可能已经有元素：
- 拉链法：挂到同桶链表里
- 开放寻址：往后探测其他空位

所以“计算位置”只是初始桶，不一定是最终物理位置。

---

## 4-68 哈希表存在元素 C，但当前线程 `get()` 不到，为什么；不加锁怎么解决

### 可能原因
- 写线程更新未对读线程可见
- rehash 中读到旧桶
- 节点插入非原子导致链表断裂
- 内存重排序

### 无锁解决
- 使用原子发布
- acquire/release 保证可见性
- RCU / hazard pointer
- 无锁哈希表算法

---

## 4-69 `hash_map` 哈希冲突解决方式有哪些

- 拉链法
- 线性探测
- 二次探测
- 双重哈希
- Robin Hood hashing（进阶）

---

## 4-70 `map` 是如何实现的，查找效率是多少
- 通常红黑树
- 查找 `O(log n)`

---

## 4-71 线性表的实现方法（腾讯）

### 常见实现
- 顺序表（数组 / 动态数组）
- 链表（单链表 / 双链表 / 循环链表）

### 对比
- 顺序表：随机访问强
- 链表：插删强

---

## 4-72 什么是迭代器？有什么作用
同 4-61，可复用。

---

## 4-73 简述 `vector::resize()` 流程，与 `list` 的区别（`erase` 时间复杂度）

### `resize(n)`
- 若 `n < size`：销毁多余元素
- 若 `n > size`
  - capacity 足够：追加默认构造元素
  - capacity 不够：扩容后追加

### `erase` 复杂度
- `vector::erase(pos)`：`O(n)`（后面元素搬移）
- `list::erase(pos)`：`O(1)`（已知位置节点断链）

---

## 4-74 `vector` 是否线程安全

不是。

### 多线程安全条件
- 只读通常可以
- 有写就需要同步

---

## 4-75 数组和链表顺序遍历到底哪个快

### 结论
通常数组（或 `vector`）更快。

### 原因
- 连续内存
- cache 命中高
- CPU 预取有效

链表虽然逻辑简单，但节点分散，cache miss 多。

---

## 4-76 `map/unordered_map/vector` 效率及内存消耗对比

### `vector`
- 查找：顺序 `O(n)`，若有序可二分 `O(log n)`
- 插入删除中间：`O(n)`
- 内存最紧凑，cache 最好

### `map`
- 查找/插删：`O(log n)`
- 每个节点额外有颜色、左右孩子、父指针
- 内存开销较大

### `unordered_map`
- 平均查找/插删：`O(1)`
- 需要桶数组 + 节点
- 内存通常也不小，且可能浪费桶空间

---

## 4-77 `vector` 和 `deque` 的区别

### `vector`
- 连续
- 随机访问最快
- 尾插强
- 头插弱

### `deque`
- 分段连续
- 头尾插入都强
- 随机访问也支持，但略慢于 `vector`

---

## 4-78 用迭代器修改了 `set` 的某个值，是否有问题

### 有问题
`set` 的元素是 key，本质上参与排序。  
如果你直接改 key，会破坏红黑树有序性。

实际上标准库里 `set` 迭代器解引用通常得到 `const Key&`，就是为了禁止你修改。

```cpp
std::set<int> s{1,2,3};
auto it = s.find(2);
// *it = 5; // ❌ 不允许
```

---

## 4-79 下标操作和 `at` 的区别

### `operator[]`
- 不做边界检查
- 越界是未定义行为
- `map::operator[]` 如果 key 不存在还会插入默认值

### `at()`
- 做边界检查
- 越界抛异常

```cpp
std::vector<int> v{1,2,3};
v[10];      // 未定义行为
v.at(10);   // 抛 out_of_range
```

---

# 附录一：高频总结表

## 1. `vector / list / deque`
| 容器 | 底层 | 随机访问 | 头插 | 尾插 | 中间插删 | 迭代器失效 |
|---|---|---:|---:|---:|---:|---|
| vector | 动态数组 | O(1) | 差 | 强 | O(n) | 扩容后大面积失效 |
| list | 双向链表 | O(n) | 强 | 强 | 已知位置 O(1) | 节点级失效 |
| deque | 分段连续 | O(1) | 强 | 强 | 中间一般 | 规则复杂 |

## 2. `map / unordered_map`
| 容器 | 底层 | 有序 | 查找 | 插入 | 删除 | 场景 |
|---|---|---|---:|---:|---:|---|
| map | 红黑树 | 是 | O(log n) | O(log n) | O(log n) | 范围查询、稳定复杂度 |
| unordered_map | 哈希表 | 否 | 平均 O(1) | 平均 O(1) | 平均 O(1) | 高频精确查找 |

---

# 附录二：面试回答模板

## 模板 1：`map` 和 `unordered_map` 区别
`map` 底层通常是红黑树，所以元素天然有序，查找、插入、删除都是 O(log n)，适合范围查询和有序遍历。`unordered_map` 底层是哈希表，平均查找 O(1)，但无序，最坏会退化到 O(n)。如果业务只关心 key 的快速精确查找，通常优先 `unordered_map`；如果需要 lower_bound、区间遍历或稳定复杂度，我会选 `map`。

## 模板 2：`vector` 和 `list` 区别
`vector` 是连续内存，随机访问快，缓存友好，尾插均摊 O(1)，但中间插删要搬移元素；`list` 是双向链表，随机访问慢，但已知位置插删 O(1)。不过在现代 CPU 下很多场景 `vector` 实际更快，因为链表 cache 命中率差。

## 模板 3：`vector` 扩容
`vector` 会维护 size 和 capacity，当 size == capacity 时需要扩容。扩容通常不是原地长大，而是重新申请一块更大的连续内存，把旧元素拷贝或移动过去，再析构旧元素并释放旧内存。所以扩容会导致迭代器、引用、指针失效。优化方式主要是提前 reserve，以及利用移动语义降低搬迁成本。

