# Interesting Stuff in Source

# ExplicitInit & LazyInit

> We cannot use a C++ static initializer to initialize certain globals because libc calls us before our C++ initializers run. We also don't want a global pointer to some globals because of the extra indirection.
>
> ExplicitInit / LazyInit wrap doing it the hard way.

- 封装初始化。

```cpp
// 正常初始化
template <typename Type>
class ExplicitInit {
    alignas(Type) uint8_t _storage[sizeof(Type)];

public:
    template <typename... Ts>
    void init(Ts &&... Args) {
        new (_storage) Type(std::forward<Ts>(Args)...);
    }

    Type &get() {
        return *reinterpret_cast<Type *>(_storage);
    }
};

// 懒加载初始化封装
template <typename Type>
class LazyInit {
    alignas(Type) uint8_t _storage[sizeof(Type)];
    bool _didInit;

public:
    template <typename... Ts>
    Type *get(bool allowCreate, Ts &&... Args) {
        if (!_didInit) {
            if (!allowCreate) {
                return nullptr;
            }
            new (_storage) Type(std::forward<Ts>(Args)...);
            _didInit = true;
        }
        return reinterpret_cast<Type *>(_storage);
    }
};
```

# DisguisedPtr

- 封装并隐藏一个指针的具体值，其本身也具有和指针相同的行为。
- 不清楚有什么具体作用，官方说可以`to hide it from tools like "leaks"`，没懂。
- [Obj-C 中的 DisguisedPtr](https://kingcos.me/posts/2021/disguised_ptr_in_objc/)

```cpp
template <typename T>
class DisguisedPtr {
    uintptr_t value;

    static uintptr_t disguise(T* ptr) {
        return -(uintptr_t)ptr;
    }

    static T* undisguise(uintptr_t val) {
        return (T*)-val;
    }
    
 public:
    DisguisedPtr() { }
    DisguisedPtr(T* ptr) 
        : value(disguise(ptr)) { }
    DisguisedPtr(const DisguisedPtr<T>& ptr) 
        : value(ptr.value) { }

    DisguisedPtr<T>& operator = (T* rhs) {
        value = disguise(rhs);
        return *this;
    }
    DisguisedPtr<T>& operator = (const DisguisedPtr<T>& rhs) {
        value = rhs.value;
        return *this;
    }
    
    operator T* () const {
        return undisguise(value);
    }
    T* operator -> () const { 
        return undisguise(value);
    }
    T& operator * () const { 
        return *undisguise(value);
    }
    T& operator [] (size_t i) const {
        return undisguise(value)[i];
    }
}
```

# StripedMap

- `StripedMap<T>`是`runtime`提供的一个范型表，用来存储锁或含有锁的结构。

- 简单情况下可以直接看做一个数组。
- `StripedMap<T>`存储的数据容量是固定的，移动端为8个，其他64个。
- 应用场景有：全局存储8个弱引用表`SideTable`。

```cpp
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };   // 移动端有8个表
#else
    enum { StripeCount = 64 };
#endif
    struct PaddedT {
        T value alignas(CacheLineSize);
    };
    PaddedT array[StripeCount];		// 存储数组

    // hash取下标
    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }
public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
}
```

# ptr_hash

- 从一个指针获取哈希值。

```cpp
#if __LP64__
static inline uint32_t ptr_hash(uint64_t key)
{
    key ^= key >> 4;
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    return (uint32_t)key;
}
#else
static inline uint32_t ptr_hash(uint32_t key)
{
    key ^= key >> 4;
    key *= 0x5052acdb;
    key ^= __builtin_bswap32(key);
    return key;
}
#endif
```

