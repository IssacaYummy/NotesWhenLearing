## 多继承

多继承它允许一个派生类（子类）同时从多个基类（父类）继承属性和行为。

### 基本语法：

```C++
class 派生类名 : 继承方式 基类名1, 继承方式 基类名2, ..., 继承方式 基类名n
{
    // 派生类新增成员
};
```

一个简单的例子

```C++
#include <iostream>
using namespace std;

// 第一个基类
class Animal {
public:
    void eat() {
        cout << "I can eat!" << endl;
    }
};

// 第二个基类
class Mammal {
public:
    void breathe() {
        cout << "I can breathe!" << endl;
    }
};

// 派生类 Bat 同时继承自 Animal 和 Mammal
class Bat : public Animal, public Mammal {
public:
    void fly() {
        cout << "I can fly!" << endl;
    }
};

int main() {
    Bat bat;
    bat.eat();      // 继承自 Animal
    bat.breathe();  // 继承自 Mammal
    bat.fly();      // 自己的成员

    return 0;
}
```

在这个例子中，`Bat`（蝙蝠）同时是一种 `Animal`（动物）和一种 `Mammal`（哺乳动物），因此它自然地继承了这两个基类的所有 `public` 成员。

## 虚继承

### 多继承经典问题：菱形继承(Diamond Problem)

菱形继承发生在当一个派生类通过多条路径继承自同一个类时

```C++
// 菱形继承的例子
class Base {
public:
    int data_;
};

class Derived1 : public Base {
    // 继承自 Base，拥有一份 data_
};

class Derived2 : public Base {
    // 继承自 Base，拥有另一份 data_
};

// MyClass 通过两条路径（Derived1 和 Derived2）继承自 Base
class MyClass : public Derived1, public Derived2 {
    // 此时，MyClass 内部有两份 Base 的副本，也就是两份 data_
};

int main() {
    MyClass obj;
    // obj.data_ = 5; // 错误！歧义：不知道要设置哪个 data_
    obj.Derived1::data_ = 10; // 正确，但需要明确指定
    obj.Derived2::data_ = 20; // 正确，但需要明确指定

    return 0;
}
```

`MyClass` 对象的内存布局中包含了两份 `Base` 的子对象。这导致了两个问题：

1. 二义性（Ambiguity）：当直接访问 `data_` 时，编译器无法知道用户想访问的是从 `Derived1` 来的还是从 `Derived2` 来的。
    
2. 数据冗余（Data Redundancy）：`MyClass` 对象中存储了两份 `Base` 的数据，这通常不是我们想要的逻辑。逻辑上，`Base` 应该只有一个。
    

### 解决方案：虚继承（Virtual Inheritance)

虚继承确保无论**虚基类**在继承层次中出现多少次，在派生类中都**只包含一个共享的虚基类子对象**。

```C++
class Base {
public:
    int data_;
};

// 使用 virtual 关键字进行虚继承
class Derived1 : virtual public Base {
    // ...
};

// 使用 virtual 关键字进行虚继承
class Derived2 : virtual public Base {
    // ...
};

// MyClass 现在只包含一份 Base 的副本
class MyClass : public Derived1, public Derived2 {
    // ...
};

int main() {
    MyClass obj;
    obj.data_ = 5; // 正确！没有歧义了，因为只有一份 data_
    obj.Derived1::data_ = 10; // 仍然可以，但修改的是同一个变量
    obj.Derived2::data_ = 20; // 同上，现在 obj.data_ 的值是 20

    cout << obj.data_ << endl; // 输出 20
    cout << obj.Derived1::data_ << endl; // 输出 20
    cout << obj.Derived2::data_ << endl; // 输出 20

    return 0;
}
```

### 注意：

- `virtual` 关键字和继承方式（`public`, `protected`, `private`）的顺序无关紧要，但通常写成 `virtual public` 或 `public virtual`。
    
- 虚继承解决了数据冗余和二义性的问题。
    
- 虚继承的实现通常通过虚基类表指针（vbptr） 来实现，编译器会为虚继承的类添加这个指针，它指向一个表，表中记录了到共享的虚基类子对象的偏移量。这带来了一定的内存和性能开销。
    

### 构造函数调用顺序

1. 虚基类的构造函数按照它们被继承的顺序（声明顺序）被调用，并且只调用一次。
    
2. 然后，非虚基类的构造函数按照它们被继承的顺序（声明顺序）被调用。
    
3. 最后，派生类自己的构造函数被调用。
    

示例：

```C++
class Base1 {
public:
    Base1() { cout << "Base1 Constructor" << endl; }
};
class Base2 {
public:
    Base2() { cout << "Base2 Constructor" << endl; }
};
class VBase {
public:
    VBase() { cout << "Virtual Base Constructor" << endl; }
};

class Derived : public Base2, virtual public VBase, public Base1 {
public:
    Derived() { cout << "Derived Constructor" << endl; }
};

int main() {
    Derived d;
    return 0;
}
```

输出：

```Plain
Virtual Base Constructor
Base2 Constructor
Base1 Constructor
Derived Constructor
```

1. 首先调用虚基类 `VBase` 的构造函数（尽管它在声明顺序中是第二个）。
    
2. 然后按照声明的继承顺序 `Base2`, `Base1` 调用非虚基类的构造函数。
    
3. 最后调用派生类自己的构造函数。
    

析构函数调用顺序正好与构造函数相反

使用时注意：

1. 接口继承：多继承的一个非常良好且常见的用法是继承多个纯抽象类（接口）。
    

```C++
// 接口类（全是纯虚函数）
class Drawable {
public:
    virtual void draw() const = 0;
    virtual ~Drawable() {} // 虚析构函数必不可少
};

class Clickable {
public:
    virtual void onClick() = 0;
    virtual ~Clickable() {}
};

// Button 是一个可绘制也是一个可点击的控件
class Button : public Drawable, public Clickable {
public:
    void draw() const override { /* 实现 */ }
    void onClick() override { /* 实现 */ }
};
```

这种用法非常清晰，因为没有数据成员，几乎不会遇到菱形继承的问题。Java 和 C# 中的“接口”就是这种理念。

2. “菱形”结构使用虚继承：如果明确要设计一个菱形继承结构（例如，`InputStream`, `OutputStream` 都继承自 `IOStreamBase`，而 `IOStream` 又同时继承 `InputStream` 和 `OutputStream`），那么应该对顶层的基类使用虚继承。
    
3. 清晰的命名和设计：如果多个基类有同名函数，确保这就是你想要的，并且使用 `::` 运算符来消除歧义，或者在新类中重写该函数来提供一个统一的接口。