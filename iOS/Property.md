# Property

# 本质

- `@property = ivar + getter + setter`

# 修饰符

## 原子性：atomic、nonatomic

- 使用`atomic`，`setter`和`getter`方法执行时会被加锁，确保原子性。

```cpp
// 具体加锁逻辑，见 objc_getProperty 
spinlock_t& slotlock = PropertyLocks[slot];
slotlock.lock();
id value = objc_retain(*slot);
slotlock.unlock();

// 和 reallySetProperty
if (!atomic) {
    oldValue = *slot;
    *slot = newValue;
} else {
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    oldValue = *slot;
    *slot = newValue;        
    slotlock.unlock();
}
```

> `setter`和`getter`方法只有在需要的时候会使用`objc_get/setProperty`，普通情况下是直接使用对象指针偏移操作读写，[勉强详解]([OC类的原理之属性的底层实现 - 掘金](https://juejin.cn/post/6976149521080418312))。

- 加锁后会有性能消耗。且仅对读写加锁，在大部分场景下仍无法保证业务的线程安全。所以在iOS中一般使用`nonatomic`。

- macOS中不用在意这些性能消耗，可以使用`atomic`。

- 默认是`atomic`。

## 读写属性

### readwrite、readonly

- `readonly`不提供`setter`。

- 默认`readwrite`。

### getter、setter

- `@property (assign, getter=isHappy) BOOL happy;`

- 显式设置属性的读写方法，不常用。

## 内存管理

### assign

- 值类型只能被`assign`修饰，读写方法中直接值类型传递。

- 引用类型如果被`assign`修饰，读写方法中对内存数据进行类型转换，会带上`__unsafe_unretained`，在内存数据被销毁时，指针不会被置空，而是变成一个野指针：

```objectivec
@property (nonatomic, assign) NSObject *c;

static NSObject * _I_AAA_c(AAA * self, SEL _cmd) { 
    return (*(NSObject *__unsafe_unretained *)((char *)self + OBJC_IVAR_$_AAA$_c)); 
}

static void _I_AAA_setC_(AAA * self, SEL _cmd, NSObject *c) { 
    (*(NSObject *__unsafe_unretained *)((char *)self + OBJC_IVAR_$_AAA$_c)) = c; 
}
```

- 野指针访问会抛`BAD_ACCESS`异常，所以不可以用`assign`修饰引用类型。

### strong

- 值类型不可以被`strong`修饰，编译报错。

- 引用类型被`strong`修饰后，读写方法中对内存数据进行的类型转换，会带上`__strong`：

- `__strong`的具体原理，见[Ownership](Ownership.md)一节。

```objectivec
@property (nonatomic, strong) NSObject *d;

static NSObject * _I_AAA_d(AAA * self, SEL _cmd) { 
    return (*(NSObject *__strong *)((char *)self + OBJC_IVAR_$_AAA$_d)); 
}

static void _I_AAA_setD_(AAA * self, SEL _cmd, NSObject *d) { 
    (*(NSObject *__strong *)((char *)self + OBJC_IVAR_$_AAA$_d)) = d; 
}
```

### copy

- 值类型不可以被`copy`修饰，编译报错。

- 引用类型被copy修饰后，读方法与被`strong`修饰相同，但是写方法使用`objc_setProperty`处理，在内部调用`copyWithZone`或`mutableCopyWithZone`执行拷贝操作：

```objectivec
@property (nonatomic, copy) NSObject *c;

static NSObject * _I_AAA_c(AAA * self, SEL _cmd) { 
    return (*(NSObject *__strong *)((char *)self + OBJC_IVAR_$_AAA$_c)); 
}

static void _I_AAA_setC_(AAA * self, SEL _cmd, NSObject *c) { 
    objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct AAA, _c), (id)c, 0, 1); 
}

static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    // ...
    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }
    // ...
}
```

### weak

- 详见[Weak](Weak.md)一节。

## 类属性：class

- `@property (nonatomic, assign, class) NSInteger classIndex;`

- 使用`class`修饰符的属性，系统不会生成变量与读写方法。
  
  - 需要自己写`static`变量和读写方法。
  
  - 或者在确保运行期间会存在读写方法后，使用`@dynamic`来告知编译器不要报警。

## 其他关键字

### synthesize

- `@synthesize name = _name;`

- 显示指定属性的变量名。

- 一般只用于协议中定义的没有变量的属性。

### dynamic

- `@dynamic name;`

- 告知编译器该属性的读写方法在运行期间是保证存在的，不要去自动生成，也不要在没有读写方法的时候报警。

- 基本用不上。
  
  - 首先你如果想自定义读写方法，会直接覆盖，不用加`@dynamic`。
  
  - 也不会有场景故意让编译器不生成读写方法，然后自己在运行期间动态生成。

# 数据存储

### 存储结构

- 一个属性的信息使用`property_t`结构存储，本类的属性最后存于`class_ro_t.baseProperties`的属性列表中。

```cpp
struct property_t {
    // 属性名
    const char *name;
    // 记录该属性的特性
    // @property (nonatomic, copy) NSString *some;
    // T@"NSString",C,N,V_some
    // Type Encodings https://nshipster.cn/type-encodings/
    const char *attributes;
};
```

- 属性生成的读写方法，同本类其他方法一样使用`method_t`结构存储，最后存于`class_ro_t.baseMethodList`的方法列表中。详见[Method](Method.md)一节。

```cpp
struct method_t {
    // 方法名，objc_selector结构，等同于字符串
    SEL name;
    // 方法类型编码
    const char *types;
    // 函数指针
    IMP imp;
};
```

- 本类属性生成的变量，使用`ivar_t`结构存储，最后存于`class_ro_t.ivars`的变量列表中。

```cpp
struct ivar_t {
    unsigned long int *offset;  // pointer to ivar offset location
    const char *name;    // 变量名字符
    const char *type;    // 类型编码
    unsigned int alignment;
    unsigned int size;
};
```

### 二进制大小

- 按`arm64`架构计算。

- 基本信息结构，存于Mach-O文件中的`__DATA、__objc_const`段，包含以下数据：
  
  - 1个`property_t`，16字节。
  
  - 读写方法2个`method_t`，48字节。
  
  - 1个`ivar_t`，28字节？

- 字符串常量，存于`__TEXT、__objc_methname`，`__TEXT、__cstring`等段中，长度视内容而定。

- 读写方法函数体，存于`__TEXT、__text`段中，视内容而定。