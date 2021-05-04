# ARM 64 汇编

> *“我几次朝夜幕伸出手去，指尖毫无所触，那小小的光点总是同指尖保持着一点不可触及的距离。”*

## 寄存器

- 31个通用整型寄存器`r0 ~ r30`，每个64位（8字节）大小。使用`x0 ~ x30`访问全64位，使用`w0 ~ w30`访问低32位：

![1.png](https://blog.cnbluebox.com/images/arm64-start/1.png)

- `x0 ~ x7`用于传递参数和函数返回值，返回值一般存于`x0`。

- `x8`是间接返回值的地址，当返回值过大时，存于`x8`地址指向的空间。
- `x9 ~ x15`是调用者保存寄存器，常用于保存本地变量。
- `x16 ~ x17`是内部调用临时寄存器。
- `x18`是平台保留寄存器，操作系统自身使用。
- `x19 ~ x28`是被调用者保存寄存器。
- `fp`（frame pointer）即`x29`，记录当前程序段的栈底地址。

- `lr`（link register）即`x30`，记录子函数返回时应该执行的指令地址。

- `sp`（stack pointer）即`x31`，记录程序段的栈顶。
- `xzr/wzr`，零寄存器，也是`x31`，使用该名称取时得0。

- 32个向量寄存器`v0 ~ v31`用于浮点计算，每个128位（16字节）大小，使用`Bn Hn Sn Dn Qn`来访问不同的位数：

![img](https://user-gold-cdn.xitu.io/2019/4/10/16a0767e95cd5e8a?imageslim)

- `pc`（program counter），记录正在执行的指令地址。

## 栈

栈即一段函数运行时存储临时变量的内存空间。函数的调用操作基于内存中栈的变化，栈的变化则表现在寄存器`fp`，`lr`，`sp`的行为上。

栈是从高地址到低地址的，**栈底为高地址**，**栈顶为低地址**。

来看一段简单的程序：

```c++
static int c() {
    return 2;
}

static void b() {
    int a = 5;
    int b = a - 5;
    b -= c();
}

static void a() {
    b();
}

int main(int argc, char * argv[]) {
    a();
}
```

这段程序使用Xcode在iOS上运行，执行的汇编如下：

```assembly
iOSTest`main:
    0x100e9e2dc <+0>:  sub    sp, sp, #0x20             ; =0x20 
    0x100e9e2e0 <+4>:  stp    x29, x30, [sp, #0x10]
    0x100e9e2e4 <+8>:  add    x29, sp, #0x10            ; =0x10 
    0x100e9e2e8 <+12>: stur   w0, [x29, #-0x4]
    0x100e9e2ec <+16>: str    x1, [sp]
    0x100e9e2f0 <+20>: bl     0x100e9e304               ; a at main.m
->  0x100e9e2f4 <+24>: mov    w0, #0x0
    0x100e9e2f8 <+28>: ldp    x29, x30, [sp, #0x10]
    0x100e9e2fc <+32>: add    sp, sp, #0x20             ; =0x20 
    0x100e9e300 <+36>: ret    
iOSTest`a:
    0x100e9e304 <+0>:  stp    x29, x30, [sp, #-0x10]!
    0x100e9e308 <+4>:  mov    x29, sp
    0x100e9e30c <+8>:  bl     0x100e9e318               ; b at main.m
->  0x100e9e310 <+12>: ldp    x29, x30, [sp], #0x10
    0x100e9e314 <+16>: ret    
iOSTest`b:
    0x100e9e318 <+0>:  sub    sp, sp, #0x20             ; =0x20 
    0x100e9e31c <+4>:  stp    x29, x30, [sp, #0x10]
    0x100e9e320 <+8>:  add    x29, sp, #0x10            ; =0x10 
    0x100e9e324 <+12>: mov    w8, #0x5
    0x100e9e328 <+16>: stur   w8, [x29, #-0x4]
    0x100e9e32c <+20>: ldur   w8, [x29, #-0x4]
    0x100e9e330 <+24>: sub    w8, w8, #0x5              ; =0x5 
    0x100e9e334 <+28>: str    w8, [sp, #0x8]
    0x100e9e338 <+32>: bl     0x100e9e354               ; c at main.m
->  0x100e9e33c <+36>: ldr    w8, [sp, #0x8]
    0x100e9e340 <+40>: sub    w8, w8, w0
    0x100e9e344 <+44>: str    w8, [sp, #0x8]
    0x100e9e348 <+48>: ldp    x29, x30, [sp, #0x10]
    0x100e9e34c <+52>: add    sp, sp, #0x20             ; =0x20 
    0x100e9e350 <+56>: ret   
iOSTest`c:
    0x100e9e354 <+0>: orr    w0, wzr, #0x2
    0x100e9e358 <+4>: ret    
```

在程序执行`main`函数之前，假设栈空间的情况如下（一个空格长8个字节，64位）：

<img src="https://raw.githubusercontent.com/fater32/Blog/master/Pictures/ARM_64_Assembly/01.png" alt="img" style="zoom:50%;" />

`fp`指向栈底高地址，`sp`指向栈顶低地址。此时进入`main`函数段，首先要为`main`开辟栈空间：

```assembly
0x100e9e2dc <+0>:  sub    sp, sp, #0x20       
0x100e9e2e0 <+4>:  stp    x29, x30, [sp, #0x10]
0x100e9e2e4 <+8>:  add    x29, sp, #0x10 
```

1. 将`sp`向低地址移动32个字节。
2. 将原先的`fp`，`lr`存至`sp+0x10`的位置。
3. 将`fp`赋值为`sp+0x10`。

此时的栈空间情况如下：

<img src="https://raw.githubusercontent.com/fater32/Blog/master/Pictures/ARM_64_Assembly/02.png" alt="img" style="zoom:50%;" />

`fp`一直存储着当前函数段的栈底，此时则指向`main`函数段的栈底位置。初始栈底`fp (origin)`与`main`函数结束后的返回地址`lr (origin)`则紧邻存储在`main`栈底之后。`fp (main)`于`sp`之间的空间就是系统为`main`函数分配的栈内存。

```assembly
0x100e9e2e8 <+12>: stur   w0, [x29, #-0x4]
0x100e9e2ec <+16>: str    x1, [sp]
```

`stur`于`str`都是存值操作，但`stur`一般后跟负地址运算。系统将`main`函数第一个参数存至`fp`减4个字节的位置，将第二个参数存至`sp`的位置，此时栈空间如下：

<img src="https://raw.githubusercontent.com/fater32/Blog/master/Pictures/ARM_64_Assembly/03.png" alt="img" style="zoom:50%;" />

```assembly
0x100e9e2f0 <+20>: bl     0x100e9e304
```

之后，就是`main`函数跳转到`a`函数的过程。`bl`指令跳转到`a`函数的位置，并且将`pc+4`，即`main`函数的吓一条指令存入`lr`，作为`a`函数的返回地址。

```assembly
0x100e9e304 <+0>:  stp    x29, x30, [sp, #-0x10]!
0x100e9e308 <+4>:  mov    x29, sp
0x100e9e30c <+8>:  bl     0x100e9e318               ; b at main.m
```

进入`a`函数后，还是和`main`相同，首先将`sp`减1个字节，然后将`fp`，`lr`存入栈，最后为`fp`赋予`a`函数的栈底。此时的栈空间如下：

<img src="https://raw.githubusercontent.com/fater32/Blog/master/Pictures/ARM_64_Assembly/04.png" alt="img" style="zoom:50%;" />

因为`a`函数并不需要存储变量，所以`fp (a)`与`sp`此时指向同一位置。紧接着使用`bl`指令跳`b`函数，同理此时系统将下一条指令存入了`lr`作为`b`函数的返回地址。

进入`b`函数后，还是先减`sp`开辟栈空间，并存`fp`和`lr`。

```assembly
0x100e9e318 <+0>:  sub    sp, sp, #0x20             
0x100e9e31c <+4>:  stp    x29, x30, [sp, #0x10]
0x100e9e320 <+8>:  add    x29, sp, #0x10 
0x100e9e324 <+12>: mov    w8, #0x5
0x100e9e328 <+16>: stur   w8, [x29, #-0x4]
0x100e9e32c <+20>: ldur   w8, [x29, #-0x4]
0x100e9e330 <+24>: sub    w8, w8, #0x5              ; =0x5 
0x100e9e334 <+28>: str    w8, [sp, #0x8]
0x100e9e338 <+32>: bl     0x100e9e354
```

入栈操作处理完后，系统开始执行`b`函数的逻辑，这其中存在一些未经优化的逻辑，总之在执行后，`sp+0x8`之后的4个字节存有`0`，`sp+0xc`的位置存有`5`。此时栈空间如下：

<img src="https://raw.githubusercontent.com/fater32/Blog/master/Pictures/ARM_64_Assembly/05.png" alt="img" style="zoom:50%;" />

跳转进入`c`函数：

```assembly
0x100e9e354 <+0>: orr    w0, wzr, #0x2
0x100e9e358 <+4>: ret
```

因为`c`函数为叶函数，所以不需开辟栈空间存储`fp`与`lr`。`orr`指令将`wzr`寄存器中的值与`2`做或运算，存至`w0`寄存器返回至`b`函数，即：

```c++
w0 = 0 | 2;
```

之后直接返回至`lr`中指向的`b`函数中跳转之后的指令`0x100e9e33c`：

```assembly
0x100e9e338 <+32>: bl     0x100e9e354               ; c at main.m
0x100e9e33c <+36>: ldr    w8, [sp, #0x8]
0x100e9e340 <+40>: sub    w8, w8, w0
0x100e9e344 <+44>: str    w8, [sp, #0x8]
```

系统取出`b`函数栈空间中存储的`0`存至`w8`，执行`w8 -= 2`，并将结果存至栈空间。因为`b`函数并没有返回值，所以这里存一下其实也没什么用。之后，就是关键的出栈操作了：

```assembly
0x100e9e348 <+48>: ldp    x29, x30, [sp, #0x10]
0x100e9e34c <+52>: add    sp, sp, #0x20             ; =0x20 
0x100e9e350 <+56>: ret   
```

系统从栈空间中，获取之间存储的`a`函数的栈空间栈底`fp (a)`和`b`函数返回`a`函数的指令地址`lr (a)`，存至`fp`，`lr`。接着将`sp`增加32个字节，不再管理`b`函数的栈空间。最后执行`ret`指令，跳转回`lr`指向的`a`函数逻辑。

进入`a`函数后，同理，使用`fp (main)`和`lr (main)`更新`fp`、`lr`，将`sp`加上之前减去的偏移，“抛弃”`a`函数的栈空间，接着`ret`跳转回`main`函数。

在`main`函数中使用`fp (origin)`和`lr (origin)`更新`fp`、`lr`，将`sp`加上之前减去的偏移，跳转回系统栈初始位置。至此，系统栈空间状态回归至了它开始的样子。

