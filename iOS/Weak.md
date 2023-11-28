# Weak

# 使用方式

- `__weak NSObject *obj = otherObj;`
- `@proprty (nonatomic, weak) NSObject *obj;`
    - 转换为cpp代码，可以发现`weak`属性也只是在读写方法实现中加上了`__weak`：

```c++
static NSObject * _I_AAA_obj(AAA * self, SEL _cmd) { return (*(NSObject *__weak *)((char *)self + OBJC_IVAR_$_AAA$_obj)); }

static void _I_AAA_setObj_(AAA * self, SEL _cmd, NSObject *obj) { (*(NSObject *__weak *)((char *)self + OBJC_IVAR_$_AAA$_obj)) = obj; }
```

- 从[Ownership](Ownership.md)一节可以看出，`__weak`宏会被`clang`转换为`__attribute__((objc_ownership(weak)))`，然后在llvm中间语言中增加调用`objc_initWeak`与`objc_destroyWeak`方法管理属性。
- `objc_initWeak`与`objc_destroyWeak`方法最终都是调用的`storeWeak`方法，通过修改OC的弱引用表来管理弱引用关系。

# 自底向上结构说明

- OC弱引用表的管理关系如下：

```cpp
storeWeak() 		// 弱引用表管理方法
-> SideTablesMap	// 静态变量，记录所有的SideTable
-> SideTable 		// 对单个weak_table_t的线程安全封装
-> weak_table_t		// 持有多个weak_entry_t
-> weak_entry_t		// 维护引用对象到弱引用指针的一对多关系
```

## weak_entry_t

- 一个对象，可以被多个弱引用指针引用。`weak_entry_t`就负责这个一对多关系的记录。

```cpp
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;	
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t out_of_line_ness : 2;		// 通过第2个位置的低2位判断是用的哪个联合体实现
            uintptr_t num_refs : PTR_MINUS_2; 	// 持有的弱引用数量，不等于已申请的长度
            uintptr_t mask;  // 已申请长度-1
            uintptr_t max_hash_displacement; // 最大hash重置次数，上层操作时用到
        };
        struct {
            // 4缓存位
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```

- 弱引用关系的记录采用联合体，弱引用不超过4个时采用数组存储，否则采用指针偏移的方式。
- `DisguisedPtr`的说明可见[Interesting Stuff in Source](Interesting Stuff in Source.md)，作用是封装指针值，其行为等同于一个指针。
    - 这里有一个点，`DisguisedPtr`封装指针后，其低2位必然是`00`或`11`。
    - 所以`weak_entry_t`通过判断`out_of_line_ness`是否等于`10`来决策本一对多关系表用的是4长度数组实现还是指针实现。

```cpp
#define REFERRERS_OUT_OF_LINE 2

bool out_of_line() {
	return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
}
```

## weak_table_t

- 一个`weak_table_t`持有多个`weak_entry_t`。

```cpp
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t num_entries;		// 有效entry数量
    uintptr_t mask;	 		// 已申请总长度
    uintptr_t max_hash_displacement;	   // 最大哈希重置次数 
};
```

- 通过`weak_entry_for_referent`方法哈希查找`weak_entry_t` ：

```cpp
static weak_entry_t *weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    ASSERT(referent);
    weak_entry_t *weak_entries = weak_table->weak_entries;
    if (!weak_entries) return nil;
	// 计算哈希值
    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    // 循环匹配
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;	// &运算可以确保结果不大于mask
        // 哈希错误
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            // 哈希重置次数超过最大次数，返回nil
            return nil;
        }
    }
    return &weak_table->weak_entries[index];
}
```

## SideTable

- `SideTable`是对`weak_table_t`的线程安全封装，持有一个`weak_table_t`和一个互斥锁`spinlock_t`：

```cpp
struct SideTable {
    spinlock_t slock;			// 名字像是自旋锁，但实现是互斥锁mutex_tt
    RefcountMap refcnts;
    weak_table_t weak_table;	// 弱引用表

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }
};
```

## SideTablesMap

- 系统使用`SideTablesMap`全局静态变量记录所有的`SideTable`，其本质是一个`StripedMap`范型结构：

```cpp
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;
```

- `ExplicitInit<T>`是`runtime`对变量的初始化逻辑封装，可直接按类型`T`使用。
- `StripedMap<T>`是`runtime`提供的一个范型表，可以直接视为类型`T`的数组。
- `ExplicitInit<T>`和`StripedMap<T>`详解都可以见[Interesting Stuff in Source](Interesting Stuff in Source.md)一节。
- 简单来说`SideTablesMap`就是一个存有8个`SideTable`的数组。

# 自顶向下逻辑说明

- 首先要明确一个逻辑：A弱引用B，是在B的弱用表上添加上A。
- 所以A弱用的对象从B改为C时，会涉及两个部分：
    - 在B的弱引用表上移除A。
    - 在C的弱引用表上添加A。

## storeWeak

- weak方面的总逻辑，和上面说的相同。
- 移除旧值弱引用使用`weak_unregister_no_lock`方法。
- 添加新值弱引用使用`weak_register_no_lock`方法。

```cpp
template <HaveOld haveOld, 
		  HaveNew haveNew, 
		  enum CrashIfDeallocating crashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj)
{
	// ...
	// 获取旧值和新值的弱引用表
	id oldObj;
    SideTable *oldTable;
    SideTable *newTable;
retry:
	if (haveOld) {
        // 存在旧值，获取旧值对应总表
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        // 存在新值，获取新值对应总表
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
	// 加锁
	SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
	// ...
	// 清理旧值的弱引用表数据
	if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
	// 在新值的弱引用表中添加数据
	if (haveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, (id)newObj, location, crashIfDeallocating ? CrashIfDeallocating : ReturnNilIfDeallocating);
        // 如果newObj不是Tagged指针，
        // 则对newObj对应的弱引用表添加SIDE_TABLE_WEAKLY_REFERENCED标识
        if (!newObj->isTaggedPointerOrNil()) {
            newObj->setWeaklyReferenced_nolock();
        }
        // 弱引用赋新值
        *location = (id)newObj;
    }
	// ...
	// 解锁
	SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
	callSetWeaklyReferenced((id)newObj);
    return (id)newObj;
}
```

## 旧引用移除

### weak_unregister_no_lock

- 移除弱引用表中的一个弱引用关系。

```cpp
void weak_unregister_no_lock(weak_table_t *weak_table,
							 id referent_id,
							 id *referrer_id)
{
    objc_object *referent = (objc_object *)referent_id;		// 被引用对象
    objc_object **referrer = (objc_object **)referrer_id;	// 弱引用指针

    weak_entry_t *entry;

    if (!referent) return;
	// 查找对应的weak_entry_t
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        // 移除referrer对referent的弱引用
        remove_referrer(entry, referrer);
        // 查找referent是否已没有弱引用
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        } else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }
        // 不再存在任何弱引用，移除referent对应的弱引用表
        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }
}
```

### remove_referrer

- 从一个`weak_entey_t`中移除一个弱引用。

```cpp
static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    if (!entry->out_of_line()) {
        // 没有超数量，还是4成员数组存储
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == old_referrer) {
                entry->inline_referrers[i] = nil;
                return;
            }
        }
        // 弱引用次数未超过边界，但4个缓存位置中没有找到这个弱引用，报错
        _objc_inform("...", old_referrer);
        objc_weak_error();
        return;
    }

    // 超过了边界，和获取对象对应的弱引用表一样，用hash寻找，超过最大hash重置次数就报错。
    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
        hash_displacement++;
        if (hash_displacement > entry->max_hash_displacement) {
            _objc_inform("...", old_referrer);
            objc_weak_error();
            return;
        }
    }
    // 移除该弱引用
    entry->referrers[index] = nil;
    entry->num_refs--;
}
```

### weak_entry_remove

- 当一个`weak_entry_t`不再包含弱引用关系时，从`weak_table_t`中移除。
- 移除后，会尝试缩减`weak_table_t`的大小。

```cpp
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));

    weak_table->num_entries--;

    // 尝试缩减总表大小
    weak_compact_maybe(weak_table);
}
```

## 新引用添加

### weak_register_no_lock

- 添加一个新的弱引用关系。

```cpp
id  weak_register_no_lock(weak_table_t *weak_table, 
                          id referent_id, 
                      	  id *referrer_id, 
                          WeakRegisterDeallocatingOptions deallocatingOptions)
{
    objc_object *referent = (objc_object *)referent_id;		// 被引用对象
    objc_object **referrer = (objc_object **)referrer_id;	// 弱引用指针

    // tagged指针直接返回自身，无需特别操作
    if (referent->isTaggedPointerOrNil()) return referent_id;

    // ...被引用对象正在dealloc时的逻辑...

    weak_entry_t *entry;
    // 查找对应的 weak_entry_t
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        // 该被引用对象已有弱引用表，添加新的弱引用
        append_referrer(entry, referrer);
    }  else {
        // 被引用对象不存在 weak_entry_t，创建
        weak_entry_t new_entry(referent, referrer);
        // 尝试扩大总表
        weak_grow_maybe(weak_table);
        // 弱引用表加入总表
        weak_entry_insert(weak_table, &new_entry);
    }
    
    // 不在该函数中置换弱引用指针的值，在外部置换
    return referent_id;
}
```

### append_referrer

- 向`weak_entry_t`添加弱引用关系。

```cpp
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (!entry->out_of_line()) {
        // 未超过4个预缓存数组元素，尝试添加
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }
        // 4个缓存空间用完了，改为指针存储
        // 这里完全可以直接申请8个元素的空间，就省去了下一步的扩容。
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[i];
        }
        // 修改联合体属性
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }
    
    ASSERT(entry->out_of_line());
    // 弱引用表当前存储的弱引用数，大于已申请空间的3/4
    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
        // 扩大空间并插入弱引用
        return grow_refs_and_insert(entry, new_referrer);
    }
    // 空间足够，直接哈希添加
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != nil) {
   		// 记录本次添加时哈希重置次数
        hash_displacement++;
        index = (index+1) & entry->mask;
        if (index == begin) bad_weak_table(entry);
    }
	// 更新最大哈希次数
    if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
    }
    // 插入新的弱引用指针
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```

### grow_refs_and_insert

- 在向`weak_entry_t`增添弱引用指针时，如果容量不足走该方法扩容，然后再添加。

```cpp
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    ASSERT(entry->out_of_line());
    // 扩容2倍
    size_t old_size = TABLE_SIZE(entry);
    size_t new_size = old_size ? old_size * 2 : 8;

    size_t num_refs = entry->num_refs;
    weak_referrer_t *old_refs = entry->referrers;
    // mask属性重置为新大小-1
    entry->mask = new_size - 1;
    
    // 建立新的数组链，重置弱引用表属性
    entry->referrers = (weak_referrer_t *)
        calloc(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    // 转移数据
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);
            num_refs--;
        }
    }
    append_referrer(entry, new_referrer);
    if (old_refs) free(old_refs);
}
```

### weak_entry_insert

- 将新的`weak_entry_t`加入`weak_table_t`。

```cpp
static void weak_entry_insert(weak_table_t *weak_table, weak_entry_t *new_entry)
{
    weak_entry_t *weak_entries = weak_table->weak_entries;
    ASSERT(weak_entries != nil);
	// 计算哈希
    size_t begin = hash_pointer(new_entry->referent) & (weak_table->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_entries[index].referent != nil) {
        // 哈希重复时，循环计算，并增加最大哈希次数
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_entries);
        hash_displacement++;
    }
	// 插入新的 weak_entry_t
    weak_entries[index] = *new_entry;
    weak_table->num_entries++;
	// 重置最大哈希计算次数
    if (hash_displacement > weak_table->max_hash_displacement) {
        weak_table->max_hash_displacement = hash_displacement;
    }
}
```

## 容量调整

- `weak_entry_t`容量的增加逻辑，在上节的`grow_refs_and_insert`方法中。`weak_entry_t`没有对应的容量缩减逻辑。
- 本节主要是`weak_table_t`容量的变化。

### weak_compact_maybe 

- 判断`weak_table_t`的已申请容量不小于1024，并且使用容量不超过1/16，缩小容量至1/8.

```cpp
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
    }
}
```

### weak_grow_maybe

- 判断`weak_table_t`的使用容量已超过全部的3/4，扩容2倍。

```cpp
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);
    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}

```

### weak_resize

- 改变`weak_table_t`容量。

```cpp
static void weak_resize(weak_table_t *weak_table, size_t new_size)
{
    size_t old_size = TABLE_SIZE(weak_table);
	// 申请新空间
    weak_entry_t *old_entries = weak_table->weak_entries;
    weak_entry_t *new_entries = (weak_entry_t *)
        calloc(new_size, sizeof(weak_entry_t));
	// 重置 weak_table_t 的属性标记
    weak_table->mask = new_size - 1;
    weak_table->weak_entries = new_entries;
    weak_table->max_hash_displacement = 0;
    weak_table->num_entries = 0;  // restored by weak_entry_insert below
    if (old_entries) {
        weak_entry_t *entry;
        weak_entry_t *end = old_entries + old_size;
        // 遍历旧数据并转移
        for (entry = old_entries; entry < end; entry++) {
            if (entry->referent) {
                weak_entry_insert(weak_table, entry);
            }
        }
        // 释放旧数据
        free(old_entries);
    }
}
```

## 被引用对象释放

- `dealloc`中会调用到`weak_clear_no_lock`。

### weak_clear_no_lock

- 逻辑不复杂，找到对应`weak_entry_t`后遍历置弱引用指针为nil即可，然后从`weak_table_t`中移除该`weak_entry_t`：

```cpp
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;
	// 查找weak_entry_t
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        return;
    }

    weak_referrer_t *referrers;
    size_t count;
    // 获取弱引用指针头与弱引用数量
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    // 遍历置nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            } else if (*referrer) {
                // 这个表里的弱引用指针指向的不是该对象，报错
                _objc_inform("...", referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    // 从table中移除entry
    weak_entry_remove(weak_table, entry);
}
```

# 特点

- `weak_table_t`对于`weak_entry_t`的管理方式，是和`weak_entry_t`对弱引用关系的管理方式一样的，二者都是使用哈希存储，并且拥有相同的属性`mask`。基于此，`weak`系统里有一个宏`TABLE_SIZE`，可以同时用于计算`weak_table_t`和`weak_entry_t`已申请的长度：

```cpp
#define TABLE_SIZE(entry) (entry->mask ? entry->mask + 1 : 0)
```

- `lockTwo`静态方法采用地址判断的方式来避免死锁。
