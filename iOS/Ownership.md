# Ownership

# __strong

## 源码转换

- OC：

```objective-c
int main() {
    __strong NSObject *a = [NSObject new];
    int e = 34;
    __strong NSObject *b = a;
    e++;
    return 0;
}
```

- 转为cpp：

```cpp
// xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-15.0.0 main.m -o main.cpp

int main() {
    __attribute__((objc_ownership(strong))) NSObject *a = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
    int e = 34;
    __attribute__((objc_ownership(strong))) NSObject *b = a;
    e++;
    return 0;
}
```

- 转为`llvm`中间代码：

```cpp
// clang -arch arm64 -fobjc-arc -emit-llvm -S main.m

define i32 @main() #0 {
  //----- begin -----
  %1 = alloca i32, align 4
  %2 = alloca %0*, align 8
  %3 = alloca i32, align 4
  %4 = alloca %0*, align 8
  store i32 0, i32* %1, align 4
      
  //----- __strong NSObject *a = [NSObject new]; -----
  %5 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %6 = bitcast %struct._class_t* %5 to i8*
  %7 = call i8* @objc_opt_new(i8* %6)
  %8 = bitcast i8* %7 to %0*
  store %0* %8, %0** %2, align 8
      
  //----- int e = 34; -----
  store i32 34, i32* %3, align 4
      
  //----- __strong NSObject *b = a; -----
  %9 = load %0*, %0** %2, align 8
  %10 = bitcast %0* %9 to i8*
  %11 = call i8* @llvm.objc.retain(i8* %10) #1
  %12 = bitcast i8* %11 to %0*
  store %0* %12, %0** %4, align 8
   
  //------ e++; -----
  %13 = load i32, i32* %3, align 4
  %14 = add nsw i32 %13, 1
  store i32 %14, i32* %3, align 4
      
  //----- end -----
  store i32 0, i32* %1, align 4
  %15 = bitcast %0** %4 to i8**
  call void @llvm.objc.storeStrong(i8** %15, i8* null) #1
  %16 = bitcast %0** %2 to i8**
  call void @llvm.objc.storeStrong(i8** %16, i8* null) #1
  %17 = load i32, i32* %1, align 4
  ret i32 %17
}
```

## 结论

- `__strong`宏会被`clang`转换为`__attribute__((objc_ownership(strong)))`。
- `__attribute__((objc_ownership(strong)))`具体是做什么用的，在cpp这一层还是看不出来。
- 要转换为`llvm`中间代码，才会发现被`__attribute__((objc_ownership(strong)))`修饰的对象，在生成时，如果对象引用计数不足，会调用`@llvm.objc.retain`，再赋值给变量。对应`runtime`源码中的`objc_retain`：

```cpp
id objc_retain(id obj)
{
    if (obj->isTaggedPointerOrNil()) return obj;
    return obj->retain();
}
```

- 在函数末尾生命周期结束时，会调用`@llvm.objc.storeStrong`方法对对象赋值`null`，来进行对象释放操作。对应`runtime`源码中的`objc_storeStrong`，会间接调用`objc_release`：

```cpp
void objc_storeStrong(id *location, id obj)
{
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}

void objc_release(id obj)
{
    if (obj->isTaggedPointerOrNil()) return;
    return obj->release();
}
```

# __weak

## 源码转换

- OC：

```objective-c
int main() {
    __strong NSObject *a = [NSObject new];
    int e = 34;
    __weak NSObject *b = a;
    e++;
    return 0;
}
```

- cpp：

```cpp
int main() {
    __attribute__((objc_ownership(strong))) NSObject *a = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
    int e = 34;
    __attribute__((objc_ownership(weak))) NSObject *b = a;
    e++;
    return 0;
}
```

- `llvm`中间代码：

```cpp
define i32 @main() #0 {
  //----- begin -----
  %1 = alloca i32, align 4
  %2 = alloca %0*, align 8
  %3 = alloca i32, align 4
  %4 = alloca %0*, align 8
  store i32 0, i32* %1, align 4
      
  //----- __strong NSObject *a = [NSObject new]; -----
  %5 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %6 = bitcast %struct._class_t* %5 to i8*
  %7 = call i8* @objc_opt_new(i8* %6)
  %8 = bitcast i8* %7 to %0*
  store %0* %8, %0** %2, align 8
      
  //----- int e = 34; -----
  store i32 34, i32* %3, align 4
      
  //----- __weak NSObject *b = a; -----
  %9 = load %0*, %0** %2, align 8
  %10 = bitcast %0** %4 to i8**
  %11 = bitcast %0* %9 to i8*
  %12 = call i8* @llvm.objc.initWeak(i8** %10, i8* %11) #1
      
  //----- e++ -----
  %13 = load i32, i32* %3, align 4
  %14 = add nsw i32 %13, 1
  store i32 %14, i32* %3, align 4
      
  //----- end -----
  store i32 0, i32* %1, align 4
  %15 = bitcast %0** %4 to i8**
  call void @llvm.objc.destroyWeak(i8** %15) #1
  %16 = bitcast %0** %2 to i8**
  call void @llvm.objc.storeStrong(i8** %16, i8* null) #1
  %17 = load i32, i32* %1, align 4
  ret i32 %17
}
```

## 结论

- `__weak`宏会被`clang`转换为`__attribute__((objc_ownership(weak)))`，依然看不出作用原理。
- `llvm`中间代码中可以看出，弱引用对象在被生成时，使用的是`@llvm.objc.initWeak`，对应`runtime`源码里的`objc_initWeak`：

```cpp
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>(location, (objc_object*)newObj);
}
```

- 对象生命周期结束时，先对弱引用对象进行释放，使用的是`@llvm.objc.destroyWeak`，对应`runtime`源码中的`objc_destroyWeak`：

```cpp
void objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>(location, nil);
}
```

# __unsafe_unretained

## 源码转换

- OC：

```objective-c
int main() {
    __strong NSObject *a = [NSObject new];
    int e = 34;
    __unsafe_unretained NSObject *b = a;
    e++;
    return 0;
}
```

- cpp：

```cpp
int main() {
    __attribute__((objc_ownership(strong))) NSObject *a = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
    int e = 34;
    __attribute__((objc_ownership(none))) NSObject *b = a;
    e++;
    return 0;
}
```

- `llvm`中间代码：

```cpp
define i32 @main() #0 {
  //----- begin -----
  %1 = alloca i32, align 4
  %2 = alloca %0*, align 8
  %3 = alloca i32, align 4
  %4 = alloca %0*, align 8
  store i32 0, i32* %1, align 4
      
  //----- __strong NSObject *a = [NSObject new]; -----
  %5 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %6 = bitcast %struct._class_t* %5 to i8*
  %7 = call i8* @objc_opt_new(i8* %6)
  %8 = bitcast i8* %7 to %0*
  store %0* %8, %0** %2, align 8
      
  //----- int e = 34; -----
  store i32 34, i32* %3, align 4
      
  //----- __unsafe_unretained NSObject *b = a; -----
  %9 = load %0*, %0** %2, align 8
  store %0* %9, %0** %4, align 8
      
  //----- e++; ------
  %10 = load i32, i32* %3, align 4
  %11 = add nsw i32 %10, 1
  store i32 %11, i32* %3, align 4
      
  //----- end -----
  store i32 0, i32* %1, align 4
  %12 = bitcast %0** %2 to i8**
  call void @llvm.objc.storeStrong(i8** %12, i8* null) #1
  %13 = load i32, i32* %1, align 4
  ret i32 %13
}
```

## 结论

- `__unsafe_unretained`宏会被`clang`转换为`__attribute__((objc_ownership(none)))`。
- `__attribute__((objc_ownership(none)))`对象不会有额外的释放操作。内存释放后，会变为野指针。
