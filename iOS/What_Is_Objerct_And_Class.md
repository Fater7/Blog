# 对象和类是什么

“万物皆对象”，似乎每一套面向对象语言都有着这样一句话。Objective-C也不例外，作为一门通用，高级，面向对象的编程语言，其也秉承着万物皆对象的大体规则，以NSObject为初始基类建立了一套高效易用的Foundation框架。但是在这之下，OC究竟是如何维护这套体系的运作，就要从Objective-C Runtime及类与对象的本质说起。

## objc_object

我们知道OC中，类为Class类型，对象为id类型。在Runtime源码中，可以找到这样几句：

```c++
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_class : objc_object {
	//...
}
```

类实际上是`objc_class`结构体类型，并且继承自对象结构体`objc_object`，这说明类本身也是一种对象。`objc_object`的结构如下所示：

```c++
struct objc_object {
private:
	isa_t isa;
public:
	// 公有方法
private:
	// 私有方法
}

union isa_t {
  	isa_t() { }
	isa_t(uintptr_t value) : bits(value) { }

	Class cls;
	uintptr_t bits;
#if defined(ISA_BITFIELD)
	struct {
	ISA_BITFIELD;  // defined in isa.h
	};
#endif
};
```

`objc_object`的唯一成员变量`isa`为联合体`isa_t`类型，其结构体值通过在不同的软硬件环境下定义宏`ISA_BITFIELD`的不同内容来填充。在64位iPhone中，`ISA_BITFIELD`的定义如下：

```c++
#if SUPPORT_PACKED_ISA
# if __arm64__
#  define ISA_BITFIELD
	uintptr_t nonpointer        : 1;	\
    uintptr_t has_assoc         : 1;	\
    uintptr_t has_cxx_dtor      : 1;	\
    uintptr_t shiftcls          : 33;	\
    uintptr_t magic             : 6;	\
    uintptr_t weakly_referenced : 1;	\
    uintptr_t deallocating      : 1;	\
    uintptr_t has_sidetable_rc  : 1;	\
    uintptr_t extra_rc          : 19
```

各bit位含义如下：

|      变量名       | bit位 |                         含义                          |
| :---------------: | :---: | :---------------------------------------------------: |
|     nonpoint      |   1   | 为1时，代表isa为一个结构体。否则代表其为一个Class指针 |
|     has_assoc     |   1   |          为1时，代表该对象存在associated对象          |
|   has_cxx_dtor    |   1   |        为1时，代表该对象存在C++或ARC的析构函数        |
|     shiftcls      |  33   |          isa为结构体时，此部分代表Class指针           |
|       magic       |   6   |            通过此位判断对象是否已经初始化             |
| weakly_referenced |   1   |           为1时，代表该对象是否持有weak对象           |
|   deallocating    |   1   |               为1时，代表该对象正在析构               |
| has_sidetable_rc  |   1   | 为1时，代表该对象的引用计数过大，无法在extra_rc中保存 |
|     extra_rc      |  19   |               表示该对象超过1的引用计数               |

## objc_class

再来看一下`objc_class`的结构：

```c++
struct objc_class : objc_object {
	Class superclass;
	cache_t cache;
	class_data_bits_t bits;
	// 公有方法
}
```

`superclass`指向父类Class显而易见。而objc_class的`isa`指向的则是元类结构：

![img](http://yulingtianxia.com/resources/Runtime/class-diagram.jpg)

从图中可以看出，`superclass`是只存在于`objc_class`中的继承链，类的`superclass`指向父类，元类的`superclass`指向元类的父类。所有的类最终都继承于根类`NSObject`，根类的`superclass`则指向`nil`。

`isa`则是存在于所有`objc_object`中的从属链。类实例`isa`指向类，类的`isa`指向其元类，而所有的元类的`isa`都指向根元类（`NSObject`的元类）。

### cache_t

缓存常用方法，优化相关性能

### class_data_bits_t

```c++
struct class_data_bits_t {
	friend objc_class;
	uintptr_t bits;
private:
	// 私有方法
public:
	// 公有方法
}
```

`class_data_bits_t`只包含一个可以作位操作的`uintptr_t`位序列，当其中包含着各种标记位以及指向`objc_class`类具体内容的指针。如`objc_class`的`data()`方法，会调用`class_data_bits_t`的同名方法，位运算返回一个指向`class_rw_t`的指针：

```c++
// in objc_class
class_rw_t *data() const {
	return bits.data();
}

// in class_data_bits_t
class_rw_t* data() const {
	return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

`class_rw_t`持有`class_ro_t`，`class_ro_t`则存储Class在编译期间已经确定的信息，二者都包含Class的方法，属性，协议等信息，但存储的方式不同：

```c++
// read_only
struct class_ro_t {
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    property_list_t *baseProperties;
    
    // others
};
```

`xxx_list_t`类型全部继承于`entsize_list_tt`，其维护一个在内存中紧密排列的`xxx_t`的数组结构，并持有一个迭代器用于操作元素。

#### method_t

```c++
struct method_t {
    SEL name;			// typedef struct objc_selector *SEL; 方法名
    const char *types;	// 方法返回值与参数类型
    MethodListIMP imp;	// using MethodListIMP = IMP; 方法体指针

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};

typedef struct method_t *Method;
```

#### ivar_t

```c++
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};

typedef struct ivar_t *Ivar;
```

#### property_t 

```c++
struct property_t {
    const char *name;
    const char *attributes;
};

typedef struct property_t *objc_property_t;
```

#### protocol_t

```c++
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;
	
	// 公有方法
}

typedef struct property_t *objc_property_t;
```

持有`class_ro_t`的`class_rw_t`则为Class提供运行时的拓展能力

```c++
// read_write
struct class_rw_t {
	const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
      
    // others
}
```

`xxx_array_t`类全部继承自`list_array_tt <xxx_t, xxx_list_t>`类，该类使用联合体存储三类不同的值：

1. 空
2. 单个`xxx_list_t`的指针
3. 一个存有多个`xxx_list_t`指针的数组

类别Category的原理就是在运行时将Category持有的`xxx_list_t`指针添加到对应`class_rw_t`的`xxx_array_t`中。

