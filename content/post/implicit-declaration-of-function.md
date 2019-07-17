---
title: 函数隐式声明 implicit declaration of function
categories: [技术记录]
date: 2018-08-14 16:58:36
---

默认函数声明`Implicit declaration function`引发的血案

文件1中定义了函数，并malloc了一段内存，返回指针

```
char *func1(char *arg){
    char *c = (char*)malloc(size);
    return c;
}
```

在文件2中直接调用该函数，前提是没有显示的声明func1

```
    ...
    new_addr = func1("foo");
    print(*new_addr);// coredump
    ...
```

在func1中分配的内存地址是`0x7fd370c11008`， 指针new_addr为`0x70c11008`。

C99文档中有说明：

>If an attempt is made to modify the result of a function call or to access it after the next sequence point, the behavior is undefined.
If the expression that denotes the called function has a type that does not include a prototype, the integer promotions are performed on each argument, and arguments that have type float are promoted to double.
These are called the default argument promotions.

在64位机器上，指针的长度是8字节，隐式的函数声明会默认截断成32位，4字节，导致丢失字节，返回了非法的内存地址，引发coredump。

编译参数要添加选项`-Wmissing-prototypes`，当出现未声明的函数，会发出warning
`warning: no previous prototype for *** [-Wmissing-prototypes]`。


**参考**
1. [c - implicit declaration of function - Stack Overflow](https://stackoverflow.com/questions/6488429/implicit-declaration-of-function)
2. [The New C Standard: 6.5.2.2](http://c0x.coding-guidelines.com/6.5.2.2.html)
