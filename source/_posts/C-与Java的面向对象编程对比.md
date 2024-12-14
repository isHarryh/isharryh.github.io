---
title: C++与Java的面向对象编程对比
toc: true
categories:
  - Programming
tags: [C++, Java]
date: 2024-12-13 18:30:00
updated: 2024-12-14 13:26:00
---

C++ 与 Java 的面向对象编程具有相似性，但也有显著的区别。本文旨在帮助掌握了 Java 的人学习 C++ 面向对象，以及帮助掌握了 C++ 面向对象的人学习 Java。

## 类的定义和继承

### 成员访问控制

C++ 对类的成员提供有 `public`、`protected` 和 `private` 三种访问级别。

- `public`：**公共**权限，无访问限制。
- `protected`：**保护**权限，派生类之外不可访问。
- `private`：**私有**权限，本类之外不可访问。

<!-- more -->

> 说白了，`public` 就是家门大开，陌生人都可以进进出出；`protected` 就是只给自己和家里的子女（派生类）开门；`private` 就是只给自己开门，连子女都不给进。

Java 在上述级别之外，还增加了 `default` 级别。

- `default`：包访问权限，在自身程序包之外不可访问。

> 访问修饰符是可以**缺省**的，这种情况下：
> 
> - 在 C++ 中，类的成员的默认访问级别是 `private`，而结构体的是 `public`。
> - 在 Java 中，类的成员的默认访问级别是 `default`。

**语法示例：**

```cpp
// C++
class A {
public:
    int x;
    int fun1() { return 123; }
protected:
    int y;
    int fun2() { return 234; }
private:
    int z;
    int fun3() { return 345; }
};
```

```java
// Java
class A {
    public int x;
    protected int y;
    private int z;
    default int m;
    int n; // also default

    public int fun1() { return 123; }
    protected int fun2() { return 234; }
    private int fun3() { return 345; }
    default fun4() { return 456; }
    int fun5() { return 567; } // also default
}
```

### 继承方式

C++ 对类的继承区分 `public`、`protected` 和 `private` 三种方式（或者说类型）。

- `public`：继承自基类的成员，在派生类中的访问级别不变。
- `protected`：继承自基类的成员，在派生类中的访问级别*最高是* `protected`。
- `protected`：继承自基类的成员，在派生类中的访问级别*最高是* `private`。

> 继承方式修饰符是可以**缺省**的，这种情况下，类的成员的默认继承方式是 `private`，而结构体的是 `public`。

Java 对类的继承*并没有*区分方式，即默认全部是 `public` 继承。

**语法示例：**

```cpp
// C++
class B : public A {
    // ...
};

class C : protected A {
    // ...
};

class D : private A {
    // ...
};
```

```java
// Java
class B extends A {
    // ...
}
```

## 多态

### 多态方法

C++ 使用 `virtual` 关键字，显式声明一个多态方法（称作**虚函数**）。当且仅当一个方法被 `virtual` 关键字修饰时，才允许它在派生类中被“显式覆写”。而 Java 的所有非静态方法都默认是多态方法，*无需*用 `virtual` 显式声明即可被覆写。

> 下面讨论 C++ 基类的某个方法*没有*被 `virtual` 修饰的情形：
> 
> - 如果派生类*使用* `override` 定义了同名方法（表明了“显式覆写”的意图），会发生报错。
> - 如果派生类*不使用* `override` 定义了同名方法，此时不会报错，而是会“隐藏”基类函数。表面上这和覆写没有区别，但是这种隐藏是基于作用域的、在编译时确定的，而不是基于动态绑定的。
> 
> 因此，强烈建议覆写时使用 `override` 关键字，以便检查是否与 `virtual` 妥当地配对。

> 此外，在 Java 中也有一个装饰器 `@Override` 起到与 C++ `override` 类似的检查作用。它不是必需的，但是推荐的。

C++ 的多态方法在基类中实现或者不实现都可以。而 Java 的多态方法在基类中必须实现，除非该方法被 `abstract` 关键字修饰成为“抽象方法”。

**语法示例：**

```cpp
// C++
class A {
public:
    int fun1() { return 11; }
    int fun2() { return 22; }
    virtual int fun3() { return 33; }
    virtual int fun4() { return 44; }
};

class B : public A {
public:
    int fun1() { return 111; } // Hiding (not recommended)
    int fun2() override { return 222; }; // Error
    int fun3() { return 333; }; // Okay (but not good enough)
    int fun4() override { return 444; }; // Okay (recommended)
};
```

```java
// Java
class A {
    public int fun1() { return 11; }
    public int fun2() { return 22; }
}

class B extends A {
    // No `@Override` is okay (but not good enough)
    public int fun1() { return 111; }

    @Override // Okay (recommended)
    public int fun2() { return 222; }
}
```

### 抽象方法

在 Java 中被 `abstract` 修饰的方法成为**抽象方法**，它必须*不实现*。当一个类存在至少一个抽象方法时，该类成为一个**抽象类**。抽象类无法被实例化。

> 在继承的过程中，抽象方法就像遗传病一样，可以传承给下一代，当然也可以在中途被实现。直到所有抽象方法都被实现，才能脱离“抽象”的牢笼。

在 C++ 中也有类似“抽象方法”的设计，即**纯虚函数**。纯虚函数通过 `= 0` 来定义。当一个类存在至少一个纯虚函数时，该类成为一个“抽象类”。

与 Java 不同的是，在 C++ 中是允许给纯虚函数做默认实现的。此外，C++ 的抽象类不需要做额外标记，但是 Java 的抽象类必须由 `abstract` 标记。

**语法示例：**

```cpp
// C++
class A {
public:
    virtual int fun1() = 0;
};

// Default implementation is okay
void A::fun1() { return 666; }

class B : public A {
public:
    int fun1() override { return 777; }
};
```

```java
// Java
abstract class A {
    // Okay
    abstract public int fun1();

    // Error (default implementation is not allowed)
    abstract public int fun2() { return 888; }
}

class B extends A {
    @Override
    public int fun1() { return 999; }
}
```

## 内存管理

### 对象创建

C++ 主要有三种创建对象的方式。

1. **动态对象**：通过 `new` 关键字手动分配对象内存，并通过 `delete` 关键字手动释放。
    > 需要注意，此时得到的是对象指针（用 `->` 访问成员）而非对象（用 `.` 访问成员）。
    ```cpp
    A* a = new A();
    a->fun();
    delete a;
    ```
2. **局部对象**：仅在当前作用域内生效的对象，离开该作用域后自动销毁。
    ```cpp
    A a;
    a.fun1();
    ```
3. **临时对象**：匿名对象，表达式结束后自动销毁。以下列出了四种常见的使用场景。
    ```cpp
    // 1. Copy the temp object to `a`
    A a = A();

    // 2. Explicitly create a temp object
    A();

    // 3. Pass argument using temp object
    fun(A());

    // 4. Some operator can create temp object
    A a3 = a1 + a2;
    ```

下表总结了三种对象的特点：

|          | 临时对象     | 局部对象     | 动态对象         |
| -------- | ------------ | ------------ | ---------------- |
| 存储位置 | 栈           | 栈           | 堆               |
| 内存管理 | 自动管理     | 自动管理     | 手动管理         |
| 销毁时机 | 表达式结束后 | 作用域结束后 | 手动 `delete` 时 |
| 是否命名 | 匿名         | 命名         | 命名             |
| 生命周期 | 短期         | 短期         | 可以长期         |

> C++ 动态对象的优点是生命周期灵活且堆内存更大，但是一定记得手动释放，否则可能导致内存泄漏。

Java 是**自动内存管理**——由垃圾回收器（GC）自动管理对象的分配和释放，不需要手动管理内存。

### 对象销毁

C++ 支持显式定义**析构函数**，用于在对象销毁时释放资源（比如动态申请的内存空间）。

Java 没有析构函数，但是可以使用 `finalize()`（已过时）或实现 `AutoCloseable` 来清理资源。
