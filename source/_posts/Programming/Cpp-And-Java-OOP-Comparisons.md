---
title: C++与Java的面向对象编程串讲
toc: true
categories:
  - Programming
tags: [C++, Java]
date: 2024-12-13 18:30:00
updated: 2025-1-3 23:13:00
---

C++ 与 Java 的面向对象编程（OOP）具有相似性，但也有显著的区别。本文将介绍和对比 C++ 和 Java 这两种语言的 OOP 语法特性，尤其旨在帮助掌握了 Java 的人学习 C++ 面向对象，以及帮助掌握了 C++ 面向对象的人学习 Java。在阅读这些内容前，读者应该至少熟悉其中一种语言的基本语法。

## 类的定义和继承

### 成员访问控制

C++ 对类（*Class*）的成员提供有 `public`、`protected` 和 `private` 三种访问级别。

- `public`：**公共**权限，无访问限制。
- `protected`：**保护**权限，派生类之外不可访问。
- `private`：**私有**权限，本类之外不可访问。

<!-- more -->

> 说白了，`public` 就是家门大开，陌生人都可以进进出出；`protected` 就是只给自己和家里的子女（派生类）开门；`private` 就是只给自己开门，连子女都不给进。

Java 在上述级别之外，还增加了 `default` 级别。

- `default`：包访问权限，在自身程序包（*Package*）之外不可访问。

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

C++ 对类的继承（*Inherit*）区分 `public`、`protected` 和 `private` 三种方式（或者说类型）。

- `public`：继承自基类（*Base Class*）的成员，在派生类中的访问级别不变。
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

> 猫和狗都是“动物”，而“动物”都有“吃”这一动作。但是，猫和狗各有各的“吃”法。这就是“动物”的多态（*Polymorphic*）的一种体现。

### 多态方法

C++ 使用 `virtual` 关键字，显式声明一个多态方法（称作**虚函数**）。当且仅当一个方法被 `virtual` 关键字修饰时，才允许它在派生类中被“显式覆写”。而 Java 的所有非静态方法都默认是多态方法，*无需*用 `virtual` 显式声明即可被覆写（*Override*）。

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

### 菱形继承

> **菱形继承**指的是派生类 B 和 C 同时继承了基类 A，而最终派生类 D 又继承了 B 和 C。继承关系的拓扑示意如下：
> 
> ```txt
>    A     <-- Base class
>  /   \
> B     C  <-- Derived class
>  \   /
>    D     <-- Final derived class
> ```
> 
> 此时，如果某个字段在类 A 中被声明，那么类 D 中可能会存在分别来自类 B 和类 C 的两份独立的字段，谓之**数据冗余问题**。
> 
> 另外，如果某个可继承的方法在类 A 中被声明，那么类 D 不知道应该采用类 B 所继承的方法还是类 C 所继承的方法，谓之**二义性问题**。

以下代码演示了在 C++ 中如何引发菱形继承问题：

```cpp
#include <iostream>
using namespace std;

class A {
public:
    int value;
    void display() { cout << "Class A" << endl; }
};

class B : public A { };
class C : public A { };
class D : public B, public C { };

int main() {
    D obj;
    obj.display(); // Error
    obj.B::display(); // Okay
    obj.C::display(); // Okay

    obj.value = 10; // Error
    obj.B::value = 10; // Okay
    obj.C::value = 20; // Okay
    cout << obj.B::value << endl; // Got 10
    cout << obj.C::value << endl; // Got 20
    return 0;
}
```

为了解决菱形继承带来的问题，C++ 和 Java 分别采用了不同的解决方案。

#### C++ 虚拟继承

C++ 使用**虚拟继承**的机制，保证了最终派生类只存在一份基类的成员拷贝。虚拟继承的语法就是用 `virtual` 修饰继承方式。

以下代码演示了在 C++ 中如何使用虚拟继承来解决菱形继承问题：

```cpp
// ...

class B : virtual public A { }; // Important
class C : virtual public A { }; // Important
class D : public B, public C { };

int main() {
    D obj;
    obj.display();
    obj.value = 10;
    cout << obj.value << endl; // Got 10
    return 0;
}
```

如果基类 A 的方法是虚方法，而派生类 B 和 C 各自对它有不同的实现，那么此时需要在最终派生类 D 中手动指定到底采用谁的实现。以下代码演示了这一过程：

```cpp
#include <iostream>
using namespace std;

class A {
public:
    virtual void display() { cout << "Class A" << endl; }
};

class B : virtual public A {
public:
    void display() override { cout << "Class B" << endl; }
};

class C : virtual public A {
public:
    void display() override { cout << "Class C" << endl; }
};

class D : public B, public C {
public:
    // Explicitly choose one
    void display() override { B::display(); }
};

int main() {
    D obj;
    obj.display(); // Got "Class B"
    return 0;
}
```

#### Java 接口

Java 的类**不允许多继承**，从而在设计层面规避了菱形继承的问题。但是 Java 提供了一种**接口**（*Interface*）机制，使得我们可以利用“多接口”来实现“多继承”。

接口是一种特殊的类型，用于定义行为规范。接口中所有方法一定是 `public` 方法，并且接口的实例方法都默认是 `abstract` 方法。接口不能实例化，必须由类来实现。

以下代码演示了 Java 的多接口（不同的接口或类应该定义在不同的文件中，这里我们将其合并展示）：

```java
interface A {
    void funA();
}

interface B extends A {
    void funB();
}

interface C extends A {
    void funC();
}

// Class can implement multi interfaces
// Note that we use `implements` here
class D implements B, C {
    // Now implement funA, funB, funC
    // void funA() { ... }
    // void funB() { ... }
    // void funC() { ... }
}

// Interface can do that too
interface D extends A, B {
    void funD();
}
```

可以发现，接口机制将方法的具体“实现”（*Implementation*）给隐去了，只有方法的头部声明，即“签名”（*Signature*）被继承了下来。于是，类 D 中的 `funA` 并不会发生实现上的冲突，类 D 只需要对“需要实现的方法”进行实现即可。

> 自 Java 8 起：
> 
> - 接口可以提供静态方法（使用 `static` 修饰）。它无法被继承，只能通过接口名调用。通常我们会用静态方法作为工具方法来使用。
> - 接口可以提供默认实现（使用 `default` 修饰）。在这种情况下，仍可能造成菱形继承的二义性问题。此时无法通过编译，开发者需要手动解决冲突。示例如下：
> 
>   ```java
>   interface A {
>       default void display() {
>           System.out.println("Hello A");
>       }
>   }
> 
>   interface B {
>       default void display() {
>           System.out.println("Hello B");
>       }
>   }
> 
>   class C implements A, B {
>       @Override
>       public void display() {
>           // Explicitly choose one
>           A.super.display();
>       }
>   }
>   ```

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

## 对象生命周期

### 内存分配

C++ 主要有三种创建对象的方式。

1. **动态对象**：通过 `new` 关键字手动分配对象内存，并通过 `delete` 关键字手动释放。
    > 需要注意，此时得到的是对象指针（用 `->` 访问成员）而非对象（用 `.` 访问成员）。

    ```cpp
    A* a = new A();
    a->fun();
    delete a;
    ```
2. **局部对象**：仅在当前作用域（*Scope*）内生效的对象，离开该作用域后自动销毁。
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

### 构造函数

**构造函数**（*Constructor*）是对象在被创建时所自动调用的函数，常用于进行一些初始化的操作。它在语法上具有两个特征：

1. 构造函数的名称与其所处的类的名称是相同的；
2. 构造函数不返回任何值，因此它们没有返回类型（连 `void` 关键字都不需要）。

#### 构造函数的分类

C++ 和 Java 中都有下列三类构造函数：

1. **默认构造函数**就是无参构造函数，即“没有参数的构造函数”；
2. **参数构造函数**是具有参数的构造函数；
3. **拷贝构造函数**接收另一个同类对象的引用，从而通过另一个对象来初始化自身。

> 拷贝构造函数通常会拷贝另一个对象的所有的成员变量到自身，相当于创建了另一个对象的副本。一次合格的拷贝操作应该是“深拷贝”的，即复制过来的各种成员变量在内存引用上和原对象是没有任何关联的。

> 自 C++ 11 起，还有有一种“移动构造函数”，用于高效地从另一个右值引用转移数据。

如果类内没有定义任何构造函数（不管有没有参数），编译器会自动提供一个隐式的默认构造函数，它会将成员变量赋值为默认值。

仅在 C++ 中，如果没有定义任何拷贝构造函数（和移动构造函数），编译器会自动提供一个默认的拷贝构造函数。

**语法示例：**

```cpp
// C++
class A {
private:
    int x;
    string name;

public:
    A() {
        this->x = 0;
        this->name = "Unknown";
        cout << "Default constructor" << endl;
    }

    A(int x, const string& name) {
        this->x = x;
        this->name = name;
        cout << "Parameterized constructor" << endl;
    }

    A(const A& other) {
        this->x = other.x;
        this->name = other.name;
        cout << "Copy constructor" << endl;
    }
};
```

```java
// Java
public class A {
    private int x;
    private String name;

    public A() {
        this.x = 0;
        this.name = "Unknown";
        System.out.println("Default constructor");
    }

    public A(int x, String name) {
        this.x = x;
        this.name = name;
        System.out.println("Parameterized constructor");
    }

    public A(A other) {
        this.x = other.x;
        this.name = other.name;
        System.out.println("Copy constructor");
    }
}
```

#### 初始化列表

C++ 提供**初始化列表**这一写法来简化成员变量的初始化，但 Java 没有这种写法。

> 初始化列表跟随在构造函数签名后面，并由冒号 `:` 引导。以下代码是初始化列表的写法示例：
> ```cpp
> // Person class
> Person(string name) : name(name), age(1) {
>     // ...
> }
> ```
> 它的执行效果类似于以下代码：
> ```cpp
> // Person class
> Person(string name) {
>     this->name = name;
>     this->age = 1;
>     // ...
> }
> ```
> 初始化列表还避免了“先默认构造再赋值”的性能开销，是 C++ 中的一种高效的初始化成员变量的方法。

#### 自身构造函数委托

Java 支持在构造函数体内使用 `this(...)` 来调用（或者说**委托**）本类的其他构造函数，它必须是构造函数的第一条语句。

C++ 不能像 Java 一样在构造函数体内调用其他构造函数。但是自 C++ 11 起，支持在初始化列表中调用其他构造函数。需要注意，一旦采用了这种写法，就不能在初始化列表中有其他初始化表达式了。

**语法示例：**

```cpp
// C++
class Person {
public:
    Person(string name, int age) : Person(name) {
        this->age = age;
    }

    Person(string name) {
        this->name = name;
    }
private:
    string name;
    int age;
};
```

```java
// Java
class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this(name);
        this.age = age;
    }

    public Person(String name) {
        this.name = name;
    }
}
```

#### 基类构造函数委托

Java 支持在构造函数体内使用 `super(...)` 来调用基类的构造函数，它必须是构造函数的第一条语句。

C++ 则需要在初始化列表的开头调用它们。

**语法示例：**

```cpp
// C++
class Book {
public:
    Book(string title, int category) {
        this->title = title;
        this->category = category;
    }
protected:
    string title;
    int category;
};

class Novel : public Book {
public:
    Novel(string title) : Book(title, 123) {
    }
};
```

```java
// Java
class Book {
    protected String title;
    protected int category;

    public Book(String title, int category) {
        this.title = title;
        this.category = category;
    }
}

class Novel extends Book {
    public Novel(String title) {
        super(title, 123);
    }
}
```

### 析构函数

C++ 支持显式定义**析构函数**（*Deconstructor*），它在对象销毁时（比如对象离开作用域或者被显式删除时）被自动调用，常用于释放资源（比如动态申请的内存空间）。析构函数的名称是“波浪号`~`+类名”。

Java 没有析构函数，但是有两种方式实现类似于析构函数的功能。

1. （已过时）覆写继承自 `Object` 基类的 `finalize()` 方法。该函数会在对象被垃圾回收（GC）时调用。但是由于 Java 的 GC 的时机具有不确定性，因此该函数具有可靠性缺陷，不推荐使用。
2. 实现 `AutoCloseable` 接口（主要是实现一个 `close()` 方法）。实现了这个接口的类的对象可以被放在 `try-with-resources` 语句中，从而简洁地管理资源的释放。

> 自 Java 7 起，支持使用 `try-with-resources` 语句来自动管理资源，从而确保在程序执行完成后，资源（如文件句柄、网络连接等）能被正确地关闭。该语句需要资源对象实现 `AutoCloseable` 接口，以便在 `try` 块结束后（无论是否发生异常）自动调用资源的 `close()` 方法。

**语法示例：**

```cpp
class A {
private:
    int* data1;
    float* data2;

public:
    A() : data1(new int[100]), data2(new float[100]) {
        cout << "Object constructed" << endl;
    }

    ~A() {
        // Release resource here
        delete[] data1;
        delete[] data2;
        cout << "Resource released" << endl;
    }
};

int main() {
    {
        A obj1;
        // Case 1: Reached the end of the scope
    }

    A* obj2 = new A();
    delete obj2; // Case 2: Explicit deletion

    return 0;
}
```

```java
// Java
public class A implements AutoCloseable {
    public A() {
        System.out.println("Object constructed");
    }

    @Override
    public void close() {
        // Release resource here
        System.out.println("Resource released");
    }

    public static void main(String[] args) {
         // Use try-with-resources
        try (A obj1 = new A()) {
            // Do something here
        }
        System.out.println("End of main()");
    }
}
```
